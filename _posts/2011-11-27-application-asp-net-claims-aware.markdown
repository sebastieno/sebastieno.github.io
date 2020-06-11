---
layout: post
title: 'Application ASP.NET Claims Aware'
date: 2011-11-27
categories: federation-didentite
resume: 'Comment déléguer l''authentification d''une application ASP.NET à un STS (Security Token Service) via le SDK Windows Identity Foundation (WIF) ?'
tags: aspnet claims
---
A l'installation du SDK Windows Identity Foundation (WIF), l'utilitaire Windows Identity Foundation Federation Utility est déployé. Cet utilitaire permet de rendre une application claims-aware en quelques clics. Un addon Visual Studio est installé avec l'utilitaire.

Nous allons voir comment, grâce à cet utilitaire, déléguer l'authentification d'une application ASP.NET à un STS (Security Token Service).

## Windows Identity Foundation Federation Utility
Pour lancer l'utilitaire, cliquez droit, dans Visual Studio, sur _Add STS reference…_ à partir d'un projet ASP.NET (si l'utilitaire ne se lance pas, vous pouvez l'exécuter depuis le menu _Administration Tools_ de Windows).

![Add STS Reference](https://farm9.staticflickr.com/8343/8210262269_84a8c66714_o.png =364x108 "Add STS Reference")

Le premier écran présente les informations sur notre application. On doit y renseigner l'emplacement du fichier de configuration (qui sera modifié par l'utilitaire) et l'URL de notre application.

![Federation Utility Wizard](https://farm9.staticflickr.com/8479/8211351098_03f91ca0f4_o.png =508x384 "Federation Utility Wizard")

Le deuxième écran permet de sélectionner le STS à qui on déléguera l'authentification. Pour la déléguer à ACS (Access Control Service, STS version Cloud de Microsoft) ou à ADFS (Active Directory Federation Services, STS version On-Premise de Microsoft), on doit sélectionner l'option _Use an existing STS_ et renseigner l'adresse du fichier `FederationMetadata.xml` du STS, fichier contenant les informations de configuration.

![Federation Utility Wizard](https://farm9.staticflickr.com/8477/8210262167_8380d1864b_o.png =508x384 "Federation Utility Wizard")

Les deux écrans suivants permettent d'activer le cryptage du token SAML ou la validation par certificat.

Le dernier écran présente la liste des claims renvoyés par le STS.

Après validation, l'utilitaire va modifier le fichier de configuration de notre application.

## Web.config

Notre application déléguant l'authentification à un service externe, l'utilitaire va désactiver l'authentification.

![Authentication Mode](https://farm9.staticflickr.com/8477/8210262447_37231be6f1_o.png =627x48 "Authentication Mode")

Deux modules HTTP sont ajoutés: `WSFederationAuthenticationModule` et `SessionAuthenticationModule`.

![Http modules](https://farm9.staticflickr.com/8069/8211351170_94d3b75f93_o.png =627x91 "Http modules")

Ces modules permettent d'ajouter tous les comportements liés à la fédération d'identité de manière transparente, sans avoir à modifier l'implémentation de notre application. Ils vont notamment gérer la redirection vers le STS lorsqu'un utilisateur non authentifié accède à une page nécessitant une authentification, la déserialisation du token SAML renvoyé par le STS lorsque l'utilisateur aura fini de s'authentifier, le stockage du token SAML dans un cookie, etc.

La section `microsoft.IdentityModel` est la section qui contient toutes les informations de configuration nécessaire au Framework WIF.

![microsoft.identityModel](https://farm9.staticflickr.com/8487/8210262365_3bf08437ec_o.png =627x333 "microsoft.identityModel")

Le token SAML renvoyé par le STS contient une `audienceUri` correspondant à l'URL de l'application pour laquelle le token a été créé. La section `audienceUris` permet de spécifier la liste des URL qui seront considérées comme appartenant à notre application. Cela permet d'éviter qu'un token créé pour une application ne soit réutilisé dans notre application.

La section `federatedAuthentication` identifie le STS à qui est déléguée l'authentification et le protocole à utiliser.

La section `claimTypeRequired` définit les claims utilisés et/ou requis par notre application.

La section `isuerNameRegistry` contient la liste des STS à qui l'on accepte de déléguer notre authentification. Cette section est très importante puisqu'elle permet d'indiquer quels sont les STS de confiance. Les tokens renvoyés par les STS enregistrés dans cette section seront considérés comme valides et l'identité de l'utilisateur se basera sur le contenu de ces tokens.

En plus de modifier le fichier de configuration, l'utilitaire va générer un fichier `FederationMetadata.xml` qui nous permettra d'inscrire notre application au STS de manière simplifiée.

_On verra dans un post suivant comment inscrire une application dans ACS ou ADFS._

![Visual Studio Solution Explorer](https://farm9.staticflickr.com/8342/8210262409_5ea4eecf30_o.png =317x261 "Visual Studio Solution Explorer")

## Request Validator

Il reste une dernière tâche à effectuer avant que l'application ne soit claims-aware.

Lorsque le STS aura authentifié l'utilisateur, il renverra à l'application ASP.NET une requête d'authentification WS-Federation contenant le token SAML (sous forme XML) en HTTP POST. Le moteur ASP.NET de validation de requête n'accepte pas de recevoir du XML et lancera une exception de sécurité. Pour résoudre ce problème, il est nécessaire de désactiver la validation des requêtes sur la page recevant la requête.

![ValidateRequest false](https://farm9.staticflickr.com/8343/8210262491_57fd09f4c9_o.png =257x21 "ValidateRequest false")

Si vous utilisez ASP.NET 4, sachez qu'on peut créer sa propre logique de validation de requête. Cela permet de valider la requête renvoyée par STS tout en laissant ASP.NET valider les autres types de requête, sans avoir à désactiver complètement la validation.

Pour utiliser ce mécanisme sur une application ASP.NET 4, il faut étendre `RequestValidator` et surcharger la méthode `IsValidRequestString`. Voici un exemple de validateur :

```csharp
public class WSFederationRequestValidator : RequestValidator
{
    protected override bool IsValidRequestString(HttpContext context, string value,
               RequestValidationSource requestValidationSource, string collectionKey,
               out int validationFailureIndex)
    {
        validationFailureIndex = 0;

        if (requestValidationSource == RequestValidationSource.Form
                    && collectionKey.Equals(WSFederationConstants.Parameters.Result,
                    StringComparison.Ordinal))
        {
            SignInResponseMessage message =
                WSFederationMessage.CreateFromFormPost(context.Request)
                as SignInResponseMessage;

            if (message != null)
            {
                return true;
            }
        }

        return base.IsValidRequestString(context, value, requestValidationSource,
                       collectionKey, out validationFailureIndex);
    }
}
```

Ajoutez ensuite une référence au validateur dans le fichier de configuration:

```xml
<httpRuntime requestValidationType="MvcApplication.WSFederationRequestValidator,
       MvcApplication, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null"/>
```

## Claims Identity

L'application est maintenant claims-aware.

Ci-dessous un diagramme de séquence décrivant les interactions liées à la fédération d'identité entre l'utilisateur, le site web et le STS.

![Schémas d'interactions](https://farm9.staticflickr.com/8481/8211351378_12290cb2e1_o.png =514x619 "Schémas d'interactions")

Lorsqu'un utilisateur va naviguer vers notre application (1), le module `WSFederationAuthenticationModule` va détecter qu'un utilisateur anonyme souhaite accéder à une page nécessitant une authentification (2) et va rediriger l'utilisateur vers le STS (3). A ce-moment là, le STS prend le relais et authentifie l'utilisateur (4).

Une fois l'authentification réalisée, le STS va renvoyer le token SAML généré (5) et rediriger l'utilisateur vers notre application (6).  Le token envoyé en HTTP POST va être déserialisé, analysé et validé par le module `WSFederationAuthenticationModule` (7). L'utilisateur peut alors accéder à la page initialement demandée (8) et un cookie contenant le token SAML est créé.

A chaque fois que l'utilisateur authentifié va naviguer vers une page de l'application, le cookie contenant le token sera renvoyé dans les headers de la requête HTTP (9). Le module `SessionAuthenticationModule` va alors décrypter et valider le token (10). La page demandée par l'utilisateur est alors retournée (11).

Le screenshot de fiddler ci-dessous illustre le mécanisme des redirections :

![Fiddler](https://farm9.staticflickr.com/8483/8211351434_55c326bbfd_o.png =460x102 "Fiddler")

_vlp-ap-sor.contoso.com_ correspond à l'URL de l'application alors que _contososrv01.contoso.com_ correspond à l'URL d'ADFS.

Au sens .NET, l'identité de l'utilisateur sera une identité de type `ClaimsIdentity`, avec une authentification de type `Federation`.

![HttpContext.User.Identity](https://farm9.staticflickr.com/8479/8211351354_ddcd73f946_o.png =563x87 "HttpContext.User.Identity")

La propriété `Claims` de la classe `ClaimsIdentity` permet d'accéder à la liste des claims renvoyée par le STS.

A bientôt sur le sujet de la fédération d'identité ;)

