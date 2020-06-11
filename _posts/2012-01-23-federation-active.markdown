---
layout: post
title: 'Fédération Active - Contacter un service WCF fédéré '
date: 2012-01-23
categories: federation-didentite
resume: 'Après l''avoir sécurisé, comment contacter un service WCF en utilisant de la fédération active (WS-Trust) via le SDK Windows Identity Foundation (WIF) ?'
tags: wcf active-federation
---
Lorsque l’on fait de la fédération d’identité, deux scénarios sont possibles pour authentifier un utilisateur: la fédération passive et la fédération active.

La fédération passive consiste à utiliser le protocole WS-Federation via les mécanismes HTTP de redirection et de POST (ce scénario convient donc plutôt à des applications Web). De cette manière, l’application ne possède aucune logique d’authentification et n’est pas responsable des données d’authentification de l’utilisateur.

Avec la fédération active, l’application possède la logique de récupération des données d’authentification de l’utilisateur et en est donc responsable. Le protocole WS-Trust est utilisé pour récupérer un token auprès du STS. Ce scénario convient donc à des applications de type service Windows contactant un service web fédéré.

La première étape lors d’une fédération active est donc de récupérer un token auprès du STS. Pour cela, le framework WIF fournit, dans le namespace `Microsoft.IdentityModel.Protocols.WSTrust`, l’interface `IWSTrustContract` permettant d’envoyer un message WS-Trust au STS, via la méthode _Issue_.

## Récupération du token d’authentification

Ci-dessous le code illustrant la récupération d'un token en fédération active.

```csharp
WSTrustChannelFactory factory = new WSTrustChannelFactory(
    new UserNameWSTrustBinding(SecurityMode.TransportWithMessageCredential),
    new EndpointAddress("https://my-sts.com/v2/wstrust/13/username"));

factory.Credentials.UserName.UserName = "demo";
factory.Credentials.UserName.Password = "demo";
factory.TrustVersion = TrustVersion.WSTrust13;

IWSTrustChannelContract proxy = factory.CreateChannel();
```

On crée une factory `WSTrustChannelFactory` en lui fournissant le binding correspondant au type d'authentification à utiliser (user/password, certificat, etc.) et l’adresse du endpoint du STS correspondant au binding (dans le code précédent, on authentifie l’utilisateur via un nom d’utilisateur et un mot de passe). Il faut alors renseigner les credentials et la version de WS-Trust à utiliser puis créer le proxy.

```csharp
RequestSecurityToken rst = new RequestSecurityToken
{
    RequestType = RequestTypes.Issue,
    AppliesTo = new EndpointAddress("https://my-domain.com/My-Application"),
    KeyType = KeyTypes.Symmetric
};
```

Il est nécessaire d'instancier un `RequestSecurityToken` qui représente un token de demande. Pour recevoir un token d’authentification, il faut que le token de demande soit de type `Issue` et il faut renseigner l’URL de l’application pour laquelle le token doit être créé.

```csharp
SecurityToken token = proxy.Issue(rst);
```

La méthode `Issue` du channel permet alors de récupérer un token SAML à partir du token de demande précédemment créé.

## Appel au service Web fédéré

A chaque fois que l’application contacte le service Web fédéré, le token d’authentification va devoir être fourni dans les headers de la requête.

Pour cela, on ne va pas pouvoir utiliser le proxy généré par Visual Studio (via Add Service Reference). Mais le namespace `Microsoft.IdentityModel.Protocols.WSTrust` (du framework WIF) fournit un ensemble de méthodes d’extension qui vont nous permettre de faire ça de manière transparente.

```csharp
WS2007FederationHttpBinding binding = new WS2007FederationHttpBinding(
    WSFederationHttpSecurityMode.TransportWithMessageCredential);

ChannelFactory<ServiceReference.IService> serviceFactory =
    new ChannelFactory<ServiceReference.IService>(binding,
    new EndpointAddress("https://my-domain.com/My-Application/Service.svc"));

serviceFactory.Credentials.SupportInteractive = false;
serviceFactory.ConfigureChannelFactory();

ServiceReference.IService proxy =
    serviceFactory.CreateChannelWithIssuedToken(token);
```

Le code précédent permet de créer un proxy vers le service en incluant le token dans les headers à chaque appel, via la méthode `CreateChannelWithIssuedToken`.

Le token d’authentification ayant une date d’expiration, il est nécessaire d’ajouter un mécanisme de vérification de la validité du token au code précédant.

Bon courage !

