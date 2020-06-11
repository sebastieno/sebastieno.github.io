---
layout: post
title: 'Sécuriser un service WCF exposé sur le Service Bus d''Azure'
date: 2012-04-25
categories: azure
resume: 'Utilisons ACS pour sécuriser des services WCF On-Premise exposés via le Relying Messaging du Service Bus d''Azure.'
tags: relying-message servicebus azure wcf
---
On a vu dans un post précédent comment profiter de la fonctionnalité de Relying Messaging du Service Bus d'Azure pour exposer nos services WCF (SOAP ou OData) internes sur internet :
<a href="https://sebastienollivier.fr/blog/azure/relying-messaging-service-bus-azure/" target="_blank">https://sebastienollivier.fr/blog/azure/relying-messaging-service-bus-azure/</a>

Les services ont été configurés sans sécurité, c'est à dire que n'importe qui pourra accéder et interroger nos services. Pour sécuriser des services, le Service Bus d'Azure prévoit d'utiliser ACS (Access Control Service).

On va voir dans ce post comment sécuriser des services exposés sur le Service Bus et ensuite comment interroger ces services.

## Fonctionnement du Service Bus

A chaque création d'un namespace Service Bus (par exemple _https://sor.servicebus.windows.net_), un namespace ACS est créé, respectant le pattern _https://&lt;service_bus_name&gt;-sb.accesscontrol.windows.net_ (dans l'exemple précédent, le namespace ACS créé sera _https://sor-sb.accesscontrol.windows.net/_).

Le Service Bus utilise cet ACS, en fédération active uniquement, pour authentifier les utilisateurs.

Pour rappel, voici les deux types de fédération (tiré du post <a href="https://sebastienollivier.fr/blog/federation-didentite/federation-active/" target="_blank">https://sebastienollivier.fr/blog/federation-didentite/federation-active/</a>) :

_La fédération passive consiste à utiliser le protocole WS-Federation via les mécanismes HTTP de redirection et de POST (ce scénario convient donc plutôt à des applications Web). De cette manière, l'application ne possède aucune logique d'authentification et n'est pas responsable des données d'authentification de l'utilisateur._

_Avec la fédération active, l'application possède la logique de récupération des données d'authentification de l'utilisateur et en est donc responsable. Le protocole WS-Trust est utilisé pour récupérer un token auprès du STS. Ce scénario convient donc à des applications de type service Windows contactant un service web fédéré._

Le Service Bus utilise le claim de type `net.windows.servicebus.action` pour connaitre les actions que le compte utilisateur peut effectuer. Il existe trois actions différentes:

* Manage : Non utilisée dans le cadre du Relying Messaging du Service Bus. Cette action est entre autres utilisée pour les fonctionnalités de Queues, Subscriptions, etc. du Service Bus.
* Listen : Action permettant d'exposer un service sur le Service Bus 
* Send : Action permettant d'interroger un service exposé sur le Service Bus 

Par défaut, le namespace du Service Bus est déclaré en tant que Relying Party sur l'ACS. Un compte _owner_ est créé possédant les actions Manage, Listen et Send.

## Configurer les droits ACS

Pour accéder à l'interface de gestion d'ACS, il faut sélectionner le Service Bus puis cliquer sur _Access Control Service_.

![Azure ACS menu](https://farm9.staticflickr.com/8210/8211409114_c5a25df3d4_o.png =192x102 "Azure ACS menu")

Dans le menu, on va pouvoir accéder à la liste des comptes via le lien _Service identities_.

![Service identities](https://farm9.staticflickr.com/8344/8210320683_1ff4410d21_o.png =139x262 "Service identities")
![Service identities list](https://farm9.staticflickr.com/8345/8211409166_8b861a168a_o.png =533x161 "Service identities list")

Pour créer un nouveau compte, il faut renseigner un nom de compte, un type de credential et un intervalle de date où le compte sera actif.

![Add service identity](https://farm9.staticflickr.com/8485/8211409078_68555ec533_o.png =377x332 "Add service identity")

Il faut ensuite déclarer les claims du compte, notamment pour positionner les actions du compte. Pour cela, il faut aller sur la page des groupes de règles, via le lien _Rule groups_, et sélectionner le groupe créé par défaut pour le Service Bus.

![Rule groups](https://farm9.staticflickr.com/8205/8211409398_f4d74efafb_o.png =116x244 "Rule groups")
![Rule groups list](https://farm9.staticflickr.com/8487/8211409352_470cdb2a02_o.png =544x163 "Rule groups list")

On va rajouter une règle pour ajouter l'action _Listen_ au compte _listener_.

Dans l'encadré _If_, il faut renseigner pour qui cette règle sera appliquée.

![If](https://farm9.staticflickr.com/8489/8211409308_b3170c00b4_o.png =478x312 "If")

L'exemple ci-dessus permet d'appliquer la règle pour les comptes de service ACS dont le `nameidentifier` est égal à `listener`.

Dans l'encadré _Then_, il faut ajouter l'action souhaitée.

![Then](https://farm9.staticflickr.com/8487/8210320713_f4901fe36a_o.png =486x169 "Then")

L'exemple ci-dessus ajoute un claim de type `net.windows.servicebus.action` ayant comme valeur `Listen` au compte précédent.

Le compte créé a maintenant le droit d'exposer un service sur le Service Bus.

## Configurer les services

Pour sécuriser un service exposé sur le Service Bus, il faut modifier le Binding Relay du service :

```xml
<webHttpRelayBinding>
    <binding name="webHttpRelayBindingConfiguration">
      <security relayClientAuthenticationType="RelayAccessToken"></security>
    </binding>
</webHttpRelayBinding>
```

L'attribut `relayClientAuthenticationType` du nœud security est positionné à `RelayAccessToken`, ce qui signifie que le client devra fournir un token de sécurité pour appeler un service du Service Bus.

Si vous essayez maintenant d'accéder à votre service via un navigateur (pour un service OData), vous devriez obtenir l'erreur suivante :

![Erreur de sécurité](https://farm9.staticflickr.com/8344/8211409030_8228feaeb1_o.png =425x132 "Erreur de sécurité")

## Récupérer un token

Maintenant que le service est sécurisé, il va falloir récupérer un token pour pouvoir l'appeler. La récupération du token va se faire en fédération passive et le token sera au format SWT (Simple Web Token), format défini par défaut dans l'ACS du Service Bus.

La récupération du token va se faire en plusieurs étapes.

On va créer un WebClient pour envoyer une requête à l'ACS. La `BaseAddress` correspond à l'URL de l'ACS du Service Bus.

```csharp
WebClient client = new WebClient
{
    BaseAddress = "https://sor-sb.accesscontrol.windows.net"
};
```

Ensuite on crée une collection de type `NameValueCollection` contenant les paramètres à envoyer. On doit y renseigner le nom et le mot de passe du compte utilisateur utilisé pour s'authentifier ainsi que le scope, c'est à dire le service pour lequel on souhaite recevoir le token.

```csharp
NameValueCollection values = new NameValueCollection
{
    {"wrap_name", "owner"},
    {"wrap_password", "***"},
    {"wrap_scope", "http://sor.servicebus.windows.net/DataService"}
};
```

On va alors poster ces informations sur le endpoint WRAPv0.9 d'ACS, correspondant à l'adresse permettant d'utiliser le protocole OAuth WRAP.


```csharp
byte[] responseBytes = client.UploadValues("WRAPv0.9", "POST", values);
string token = Encoding.UTF8.GetString(responseBytes);
```

Le token est maintenant récupéré, il reste à l'envoyer dans les headers des requêtes WCF.

## Appeler un service WCF Data Services

L'ajout d'un header dans une requête WCF Data Services est plutôt simple. Il suffit de s'abonner à l'évènement `SendingRequest`.

```csharp
DataServiceContext service = new DataServiceContext(
               new Uri("https://sor.servicebus.windows.net/DataService"));
service.SendingRequest += ServiceSendingRequest;

internal static void ServiceSendingRequest(object sender, 
               SendingRequestEventArgs e)
{
    // Récupérer le token
    string securityToken = [...];

    if (!string.IsNullOrEmpty(securityToken))
    {
        string headerValue = string.Format("WRAP access_token=\"{0}\"", 
               HttpUtility.UrlDecode(securityToken));
        e.Request.Headers.Add("Authorization", headerValue);
    }
}
```

## Appeler un service WCF SOAP

L'ajout d'un header dans une requête WCF SOAP est plus compliqué.

Il faut commencer par créer un Custom Header permettant d'envoyer le token.

```csharp
public class SwtHeader : MessageHeader
{
    private readonly string token;

    public SwtHeader(string token)
    {
        this.token = token;
    }

    public override string Name
    {
        get { return "RelayAccessToken"; }
    }

    public override string Namespace
    {
        get { return "http://schemas.microsoft.com/netservices/2009/05/servicebus/connect"; }
    }

    protected override void OnWriteHeaderContents(XmlDictionaryWriter writer, MessageVersion messageVersion)
    {
        writer.WriteStartElement("wsse", "BinarySecurityToken", "http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd");
        writer.WriteAttributeString("wsu", "Id", "http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd", string.Format("uuid:{0}", Guid.NewGuid().ToString("D")));
        writer.WriteAttributeString("ValueType", "http://schemas.xmlsoap.org/ws/2009/11/swt-token-profile-1.0");
        writer.WriteAttributeString("EncodingType", "http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-soap-message-security-1.0#Base64Binary");
        writer.WriteString(Convert.ToBase64String(Encoding.UTF8.GetBytes(token)));
        writer.WriteEndElement();
    }
}
```

Ce header contient le token SWT encodé au format `BinarySecurityToken` (lien <a href="http://msdn.microsoft.com/en-us/library/microsoft.web.services2.security.tokens.binarysecuritytoken.aspx" target="_blank">MSDN</a>). Le header aura la forme suivante :

```xml
<RelayAccessToken xmlns="http://schemas.microsoft.com/netservices/2009/05/servicebus/connect">
    <wsse:BinarySecurityToken wsu:Id="uuid:7103dcc6-976a-4dd7-9101-1318ccb5907c" 
	ValueType="http://schemas.xmlsoap.org/ws/2009/11/swt-token-profile-1.0"
	EncodingType="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-soap-message-security-1.0#Base64Binary"
	xmlns:wsse="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd"
	xmlns:wsu="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-utility-1.0.xsd">
	***Encoded token***
    </wsse:BinarySecurityToken>
</RelayAccessToken>
```

Il doit être inséré dans la requête SOAP via l'utilisation de `OperationContextScope`.

```csharp
OperationContextScope scope = new OperationContextScope(client.InnerChannel);

// Récupérer le token
string securityToken = [...];

MessageHeader swtHeader = new SwtHeader(HttpUtility.UrlDecode(securityToken));
OperationContext.Current.OutgoingMessageHeaders.Add(swtHeader);
```

## Gestion de l'expiration du token

Le token SWT renvoyé par ACS a une date d'expiration (UTC), à partir de laquelle il ne sera plus possible de l'utiliser pour interroger le service. Il est donc nécessaire de prendre en compte cette notion d'expiration.

Ci-dessous le code permettant d'extraire la date d'expiration, stockée dans le token SWT sous le format <a href="http://fr.wikipedia.org/wiki/Epoch" target="_blank">EPOCH</a> :

```csharp
string[] splittedToken = requestTokenResponse
        .Split(new[] { HttpUtility.UrlEncode("&") }, StringSplitOptions.None);
string expiresOn = splittedToken.First(s => s.StartsWith("ExpiresOn"));
long epoch = Int64.Parse(expiresOn
        .Split(new[] { HttpUtility.UrlEncode("=") }, StringSplitOptions.None)
        .Last());
DateTime expirationDate = new DateTime(1970, 1, 1, 0, 0, 0, DateTimeKind.Utc)
        .AddSeconds(epoch);
```

La solution de facilité est de récupérer un nouveau token à chaque appel au service WCF. L'avantage est qu'on ne doit plus du tout se soucier de l'expiration du token, par contre on double le nombre de requêtes. La meilleure solution reste donc de stocker le token pour l'utiliser tant qu'il est valide. A son expiration, on demande un nouveau token.

Dans le but de factoriser cette gestion de l'expiration, j'ai créé un Helper qui stocke les tokens SWT jusqu'à leur date d'expiration et qui les renouvèle quand il reste moins de 5 secondes de validité (valeur arbitraire).

Techniquement, le stockage des tokens se fait dans un <a href="http://msdn.microsoft.com/fr-fr/library/dd287191.aspx" target="_blank">`ConcurrentDictionary`</a>, permettant de gérer les accès concurrents. L'implémentation du `ConcurrentDictionary` n'est pas optimum (il y a quelques problèmes de lock lors des accès simultanés en `Add`) mais il sert de bonne base pour faire un Singleton multi-threadé.

Pour récupérer un token à partir de cet Helper, il faut appeler la méthode suivante :

```csharp
/// <summary>
/// Récupère un token SWT d'ACS permettant d'interroger un service hébergé dans le Service Bus
/// </summary>
/// <param name="serviceRelativeUri">Uri relative du service pour lequel le token doit être récupéré</param>
/// <param name="serviceNamespace">Nom du namespace du Service Bus exposant le service</param>
/// <param name="issuer">Nom du compte à utiliser pour récupérer le token</param>
/// <param name="key">Clef du compte à utiliser pour récupérer le token</param>
/// <returns>Token SWT</returns>
public static string GetToken(string serviceRelativeUri, string serviceNamespace, string issuer, string key)
```

Un appel à cette méthode donnerait :

```csharp
string securityToken = SecurityTokenHelper
    .GetToken("SoapService", "sor", "owner", "*****");
```

Vous trouverez le zip contenant les fichiers du Helper ci-dessous (n'hésitez pas à ajouter vos propres modifications) : <a href="https://skydrive.live.com/redir?resid=2BAE7BB2DE0FBFC7!286" target="_blank">_Sor.ServiceBus.Security.zip_</a>

Bon courage

