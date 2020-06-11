---
layout: post
title: 'Récupération des informations utilisateurs via les Graph API de Windows Azure Active Directory'
date: 2013-04-18
categories: azure
resume: 'Découvrons la Graph API pour récupérer les informations des utilisateurs connectés depuis WAAD (Windows Azure Active Directory).'
tags: graphapi aad
---
Dans le post précédent, on a vu qu’il était possible de déléguer l’authentification de notre application à WAAD, via de la fédération passive :

<a href="https://sebastienollivier.fr/blog/azure/authentification-via-windows-azure-active-directory-production/">https://sebastienollivier.fr/blog/azure/authentification-via-windows-azure-active-directory-production/</a>

Après que l’utilisateur se soit authentifié, la liste des claims renvoyée sera la suivante (la liste peut légèrement différer en fonction de la configuration de votre WAAD) :

![Liste des claims](https://farm9.staticflickr.com/8245/8588472539_bcb4a30439_o.png =728x122 "Liste des claims")

Pour pouvoir récupérer d’autres claims (rôles, groupes, etc.), on ne va pas pouvoir passer directement par WAAD (comme on pourrait le faire en configurant ACS ou ADFS) mais on va devoir s’appuyer sur les 
<a title="Graph Api" href="http://msdn.microsoft.com/en-us/library/windowsazure/hh974476.aspx" target="_blank">Graph Api</a>. Les Graph Api sont des API REST exposant les objects Active Directory pour un tenant donné.

On va voir dans ce post comment récupérer des informations de l’utilisateur connecté depuis WAAD.

## Déclaration des droits de l’application sur WAAD

Dans le post précédent, on a dû enregistrer notre application sur WAAD, via le portail Azure.

Avant de rentrer dans le vif du sujet, il va falloir qu'on s'assure d'avoir donné les droits de lecture (voire d'écriture) sur les Graph API à notre application. Pour cela, on va retourner sur le portail, et plus précisement sur l'application que nous avons enregistrée.

Le bouton _Manage Access_ va permettre de modifier les droits d'accès de l'application :

![Manage Access](https://farm9.staticflickr.com/8106/8656694999_fbe9891a0f_o.png =298x61 "Manage Access")

Cliquez sur _Change the directory access for this app_ pour sélectionner le niveau de droit :

![Manage Access Wizard](https://farm9.staticflickr.com/8114/8657799858_72051b6de0_o.png =421x226 "Manage Access Wizard")

Vous pouvez sélectionner un droit en fonction des actions que vous souhaitez effectuer :

![Directory Access](https://farm9.staticflickr.com/8113/8656694995_a8f54da562_o.png =475x235 "Directory Access")

L'application a maintenant le droit d'accéder aux Graph API. Il faut ensuite créer une clef qui permettra d'authentifier l'application auprès des API.

Dans l'onglet _Configure_, la zone _Keys_ liste les clefs de l'application. Pour en créer une, il suffit de sélectionner une durée puis d'enregistrer :

![Application keys](https://farm9.staticflickr.com/8124/8657812008_652395a4be_o.png =682x211 "Application keys")

On peut maintenant authentifier l'application via cette clef, il ne nous reste plus qu'à requêter le service.

## Requêtage des Graph API

Comme on l'a vu, les Graph API sont des services REST. Avant de pouvoir communiquer avec eux, il va falloir récupérer un token depuis ACS pour notre tenant et notre application. Ce token devra ensuite être envoyé dans chaque requête aux Graph API.

Pour faciliter le travail, Microsoft a développé un helper qui encapsule toute la mécanique d'authentification. Vous pouvez trouver le projet ici : <a href="http://code.msdn.microsoft.com/windowsazure/Windows-Azure-AD-Graph-API-a8c72e18" target="_blank">http://code.msdn.microsoft.com/windowsazure/Windows-Azure-AD-Graph-API-a8c72e18</a>

Pour récupérer un token, il faut appeler la méthode statique `GetAuthorizationToken` de la classe `DirectoryDataServiceAuthorizationHelper` en renseignant le nom du tenant, l'id de l'application ainsi que la clef que l'on a créé précédemment.

```csharp
var token = DirectoryDataServiceAuthorizationHelper.GetAuthorizationToken(
                                 "sebastieno.onmicrosoft.com", 
                                 "http://localhost/Sor.GraphApi/",
                                 "xxx");
```

En retour de cette méthode, on va recevoir le token. A partir de là, on va pouvoir créer une nouvelle instance de la classe `DirectoryDataService` en passant le nom du tenant et le token récupéré.

```csharp
var dataService = new DirectoryDataService("sebastieno.onmicrosoft.com", token);
```

Cette classe expose un ensemble de `DataServiceQuery` représentant les différents objets AD exposés par les Graph API. Pour récupérer l'utilisateur connecté à partir de son UPN, on doit utiliser la syntaxe suivante :


```csharp
ClaimsIdentity identity = (ClaimsIdentity)HttpContext.User.Identity;

var upn = identity.Claims.First(claim => claim.Type == ClaimTypes.Name);

var user = dataService.users.Where(u => u.accountEnabled == true && u.userPrincipalName == upn.Value)
   .AsEnumerable()
   .SingleOrDefault();
```

Pour récupérer les rôles de l'utilisateur, il suffit alors d'écrire :

```csharp
var roles = dataService.LoadProperty(user, "memberOf").OfType<Role>();
```

Plutôt simple !

On pourrait créer une surcharge de `ClaimsAuthenticationManager` en overridant la méthode `Authenticate` qui irait chercher les rôles de l'utilisateur (en utilisant le code précédent) et qui les rajouterait à l'identité `HttpContext.User.Identity` en tant que claim de type `Role`. De cette manière, on pourra utiliser les attributs `Authorize` et la méthode `IsInRole` de manière classique dans toute l'application.

Bon requêtage !


