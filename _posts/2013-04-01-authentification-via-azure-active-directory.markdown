---
layout: post
title: 'Authentification via Windows Azure Active Directory Preview'
date: 2013-04-01
categories: azure
resume: 'Utilisons WAAD (Windows Azure Active Directory), en version Preview, pour sécuriser nos applications d''entreprises en déléguant l''authentification.'
tags: waad aad authentication
---
**[Update : WAAD est depuis disponible en version Production. Vous pouvez vous référer au lien suivant pour connaître les modifications : <a href="https://sebastienollivier.fr/blog/azure/authentification-via-azure-active-directory-prod" target="_blank">https://sebastienollivier.fr/blog/azure/authentification-via-azure-active-directory-prod</a>]**

Une nouvelle brique Azure est apparue il y a quelques temps : WAAD (Windows Azure Active Directory). Comme son nom l’indique, WAAD est la version cloud d’Active Directory. L’idée est d’avoir tout son annuaire d’entreprise dans le cloud (haute disponibilité, coût d’infrastructure réduit, etc.).

L’un des principaux enjeux avec WAAD est de pouvoir authentifier les applications d’entreprises via ce service. On va voir dans ce post qu’on peut facilement faire de la fédération passive avec WAAD.

_WAAD est en version Preview, ce qui veut dire que toutes les informations que vous trouverez ci-dessous sont susceptibles d’évoluer._

## Enregistrement de l’application dans WAAD

La première étape quand on fait de la fédération passive est de déclarer l’application auprès du STS. Dans le cas de WAAD, on va être obligé de passer par du PowerShell :( (ce qui n'est plus le cas pour la version Production).

Les cmdlets sont téléchargeables ici (il est nécessaire d’installer _Microsoft Online Services Sign-in Assistant_ et _Windows Azure AD Module for Windows PowerShell_) :<br /><a href="http://technet.microsoft.com/library/jj151815.aspx#BKMK_Requirements" target="_blank">http://technet.microsoft.com/library/jj151815.aspx#BKMK_Requirements</a>

Une fois installé, il va falloir exécuter 4 commandes. Les deux premières permettent de charger le module PowerShell et de déclencher la connexion à WAAD (vous serez prompté) :

```powershell
Import-Module MSOnlineExtended –Force
Connect-MsolService
```

Il va falloir ensuite déclarer l’application via la commande <a href="http://technet.microsoft.com/en-us/library/dn194119.aspx" target="_blank">`New-MsolServicePrincipal`</a>, en renseignant le ou les Service Principal Names (alias de l'application), le nom de l’application ainsi qu’une date de début et une date de fin de validité de l’application :

```powershell
New-MsolServicePrincipal -ServicePrincipalNames @("Sor.WaadAuthentificationWebSite") `
    -DisplayName "Poc Waad Authentification" ` 
    -Type Symmetric -Usage Verify ` 
    -StartDate "14/03/2013" -EndDate "14/03/2014"
```

En output de cette commande, vous recevrez tout un tas d’informations dont 3 importantes :
* Symmetric Key : Clef permettant d’authentifier l’application auprès de WAAD (notamment utilisée pour les Graph API). Cette clef ne sera visible qu’ici, donc veillez à bien la sauvegarder.
* AppPrincipalId et ObjectId : Identifiants de l’application

L’application est maintenant connue de WAAD. Il reste à déclarer les Url de retour (utilisées par WAAD pour rediriger l’utilisateur vers notre application). La commande `New-MsolServicePrincipalAddresses` permet de créer un objet représentant l’adresse de retour de l’application. La commande `Set-MsolServicePrincipal` permet d’enregistrer cette adresse :

```powershell
$replyUrl = New-MsolServicePrincipalAddresses –Address "https://localhost/Sor.WaadAuthentificationWebSite"
Set-MsolServicePrincipal –AppPrincipalId “<AppPrincipalId>” -Addresses $replyUrl
```

## Délégation de l’authentification à WAAD

La deuxième étape consiste à configurer l’application pour déléguer toute la partie authentification à WAAD.

L’installation du SDK WIF est nécessaire pour cette partie : <a href="http://www.microsoft.com/en-us/download/details.aspx?id=4451" target="_blank">http://www.microsoft.com/en-us/download/details.aspx?id=4451</a>

### Visual Studio 2010

Si vous utilisez Visual Studio 2010, le SDK WIF va installer l’option “Add STS Reference…” au clic droit sur le projet. Utilisez cette option pour lancer le wizard.

![Add STS Reference...](https://farm9.staticflickr.com/8519/8583181779_c9caeb3d43_o.png =372x125 "Add STS Reference...")

### Visual Studio 2012

Si vous utilisez Visual Studio 2012, l’option “Add STS Reference…” ne sera pas disponible. Il existe cependant une extension permettant de faire l’équivalent : <a href="http://visualstudiogallery.msdn.microsoft.com/e21bf653-dfe1-4d81-b3d3-795cb104066e" target="_blank">http://visualstudiogallery.msdn.microsoft.com/e21bf653-dfe1-4d81-b3d3-795cb104066e</a>

![Identity And Access...](https://farm9.staticflickr.com/8239/8584282874_2049179418_o.png =453x118 "Identity And Access...")

Dans ces wizards, vous allez devoir renseigner l’URL du fichier de metadata de WAAD (fichier comprenant toutes les informations nécessaires à la configuration de la fédération passive) et l’Application URI de votre application.

Comme WAAD gère chaque domaine comme un tenant, les URL vont devoir respecter la structure suivante :
* Url du fichier de metadata : https://accounts.accesscontrol.windows.net/<Domain>/FederationMetadata/2007-06/FederationMetadata.xml<br />_Ex: <a href="https://accounts.accesscontrol.windows.net/sebastieno.onmicrosoft.com/FederationMetadata/2007-06/FederationMetadata.xml" target="_blank">https://accounts.accesscontrol.windows.net/sebastieno.onmicrosoft.com/FederationMetadata/2007-06/FederationMetadata.xml</a>_
* Application Uri : spn:<AppPrincipalId>@<Domain><br />_Ex: spn:xxxxx@sebastieno.onmicrosoft.com_

Une fois validé, le wizard va modifier le fichier web.config de l’application pour configurer la fédération passive.

_Pour plus d’informations sur les modifications effectuées sur le fichier web.config : <a href="https://sebastienollivier.fr/blog/federation-didentite/application-asp-net-claims-aware/" target="_blank">https://sebastienollivier.fr/blog/federation-didentite/application-asp-net-claims-aware/</a>_

Il reste une dernière modification à effectuer avant d’avoir terminé le processus. Dans le wizard, on a fourni une Application Uri au format _&lt;AppPrincipalId&gt;@&lt;Domain&gt;_. WAAD ne pourra pas utiliser cette information pour rediriger l’utilisateur vers notre application une fois l’authentification terminée. Il est donc nécessaire de modifier le fichier web.config pour rajouter l’attribut `reply` en spécifiant l’URL de retour (la même que celle définie lors de l’enregistrement de l’application dans WAAD) :

```xml
<system.identityModel.services>
    <federationConfiguration>
        <cookieHandler requireSsl="false" />
        <wsFederation passiveRedirectEnabled="true" 
            issuer="https://accounts.accesscontrol.windows.net/sebastieno.onmicrosoft.com/v2/wsfederation"
            realm="spn:xxxx@sebastieno.onmicrosoft.com" 
            reply="https://localhost/Sor.WaadAuthentication/"
            requireHttps="false" />
    </federationConfiguration>
</system.identityModel.services>
```

La délégation d’identité est maintenant active. L’utilisateur sera redirigé vers WAAD pour s’authentifier. Une fois revenu sur notre application, pour récupérer son identité, il faudra utiliser la syntaxe suivante :

```csharp
using System.Security.Claims;

[...]

ClaimsIdentity identity = HttpContext.Current.User.Identity as ClaimsIdentity
```

![HttpContext.Current.User.Identity](https://farm9.staticflickr.com/8245/8588472539_bcb4a30439_o.png =968x149 "HttpContext.Current.User.Identity")

## ASP.NET and Web tools 2012.2 & Microsoft ASP.NET Tools for WAAD

Si vous avez la flemme de faire ces quelques étapes, Microsoft a tout prévu.

En installant _<a href="http://go.microsoft.com/fwlink/?LinkID=275131" target="_blank">ASP.NET and Web Tools 2012.2</a>_ et _<a href="http://go.microsoft.com/fwlink/?LinkID=282306" target="_blank">Microsoft ASP.NET Tools for Windows Azure Active Directory</a>_, la fonctionnalité _Enable Windows Azure Authentication_ sera disponible dans le menu _Project_.

![Enable Windows Azure Authentication...](https://farm9.staticflickr.com/8100/8588504229_4a185afd44_o.png =385x265 "Enable Windows Azure Authentication...")

Il suffit alors de renseigner le nom du domaine WAAD, et l’outil fera tout pour nous : déclaration de l’application dans WAAD et configuration de l’application (on peut aussi indiquer qu’on ne souhaite que configurer l’application).

![Wizard Enable Windows Azure Authentication...](https://farm9.staticflickr.com/8231/8589667332_e8b85c0b74_o.png "Wizard Enable Windows Azure Authentication...")

Par contre, avec cette méthode, on ne récupère pas les informations d’enregistrement de l’application (ObjectId, ApplicationId et Symmetric Key notamment). Pour les deux premières, on pourra les retrouver en fouillant dans le fichier web.config mais on n’aura pas la possibilité de récupérer la Symmetric Key. Si on souhaite par la suite utiliser des éléments la nécessitant (par exemple les Graph API), on devra au préalable en générer une nouvelle, via PowerShell.

A bientôt !

**[Update : WAAD est depuis disponible en version Production. Vous pouvez vous référer au lien suivant pour connaître les modifications : <a href="https://sebastienollivier.fr/blog/azure/authentification-via-azure-active-directory-prod" target="_blank">https://sebastienollivier.fr/blog/azure/authentification-via-azure-active-directory-prod</a>]**


