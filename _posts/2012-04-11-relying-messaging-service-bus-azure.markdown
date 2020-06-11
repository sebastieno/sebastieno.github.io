---
layout: post
title: 'Exposer un service WCF via le Service Bus Azure'
date: 2012-04-11
categories: azure
resume: 'Le Service Bus d''Azure propose le Relying Messaging permettant d''exposer des services hébergés On-Premise sur Azure. Découvrons comment cela fonctionne.'
tags: relying-message servicebus azure wcf
---
Le Service Bus apporte les fonctionnalités de messagerie et de connectivité sécurisées à Azure.

La partie Relying Messaging du Service Bus permet d'exposer des services hébergés On-Premise sur Azure, de manière totalement sécurisée, sans avoir à créer de DMZ. Le Relying Messaging prend en charge l'ensemble des protocoles de services Web.

On va voir à travers ce post comment exposer un service WCF SOAP ainsi qu'un service WCF Data Services au travers du Service Bus d'Azure.

Microsoft met à disposition la dll `Microsoft.ServiceBus.dll` pour interagir avec le Service Bus. Cette dll définit un ensemble d'éléments (binding, behavior, etc.) qui nous permettront de configurer les services WCF.

Tout va se passer dans les fichiers de configuration des services.

## Configurer un service WCF SOAP

La première configuration à effectuer est l'ajout des trois extensions suivantes, définies dans la dll `Microsoft.IdentityModel.dll` :

* TransportClientEndpointBehavior : Permet de renseigner les credentials à utiliser pour se connecter au Service Bus
* ServiceRegistrySettings : Permet de définir la visibilité du service WCF sur le Service Bus. Si la visibilité est définie sur `Public`, le service sera visible en naviguant vers l'URL du Service Bus.        
* WS2007HttpRelayBinding : Représente un binding de type WS2007HttpBinding supportant une liaison avec le Service Bus.

```xml
<extensions>
  <behaviorExtensions>
    <add name="transportClientEndpointBehavior" 
         type="Microsoft.ServiceBus.Configuration.TransportClientEndpointBehaviorElement, 
         Microsoft.ServiceBus, Version=1.6.0.0, Culture=neutral, 
         PublicKeyToken=31bf3856ad364e35"/>
    <add name="serviceRegistrySettings" 
         type="Microsoft.ServiceBus.Configuration.ServiceRegistrySettingsElement, 
         Microsoft.ServiceBus, Version=1.6.0.0, Culture=neutral, 
         PublicKeyToken=31bf3856ad364e35"/>
  </behaviorExtensions>
  <bindingExtensions>
    <add name="ws2007HttpRelayBinding" 
         type="Microsoft.ServiceBus.Configuration.WS2007HttpRelayBindingCollectionElement, 
         Microsoft.ServiceBus, Version=1.6.0.0, Culture=neutral, 
         PublicKeyToken=31bf3856ad364e35"/>
  </bindingExtensions>
</extensions>
```

On va ensuite ajouter un nouvel Endpoint Behavior:

```xml
<endpointBehaviors>
  <behavior name="sharedSecretClientCredentials">
    <serviceRegistrySettings discoveryMode="Public">
    </serviceRegistrySettings>
    <transportClientEndpointBehavior>
      <tokenProvider>
        <sharedSecret issuerName="owner" 
                      issuerSecret="**********" />
      </tokenProvider>
    </transportClientEndpointBehavior>
  </behavior>
</endpointBehaviors>
```

Ce behavior permet à la fois de déclarer la visibilité du service (via `serviceRegistrySettings`) et les credentials à utiliser pour se connecter sur le Service Bus (via `transportClientEndpointBehavior`).

Le Service Bus utilise ACS (Access Control Service) d'Azure pour gérer la sécurité. A la création d'un nouveau namespace de Service Bus, ce namespace sera associé à un namespace ACS. Par défaut, un compte nommé _owner_ sera créé. Nous allons utiliser ce compte pour connecter notre service au Service Bus. Pour récupérer la clef à renseigner dans l'attribut `issuerSecret` du noeud `sharedSecret`, il faut vous rendre sur le portail Azure, puis sélectionner le Service Bus. Dans le panneau des propriétés, vous aurez un bouton _Default Key_ qui vous permettra de la récupérer.

![Propriétés du service bus](https://farm9.staticflickr.com/8198/8211408024_79f9dd4aee_o.png =513x266 "Propriétés du service bus")

On va maintenant déclarer un nouveau binding de type `WS2007HttpRelayBinding` :

```xml
<ws2007HttpRelayBinding>
  <binding name="httpRelayEndpointConfig">
    <security relayClientAuthenticationType="None" />
  </binding>
</ws2007HttpRelayBinding>
```

Ce binding permet de définir si on souhaite sécuriser le service avec ACS (via `relayClientAuthenticationType`). La sécurité sera désactivée sur ce post, on en parlera dans un suivant.

La dernière étape consiste à rajouter un endpoint à notre service, l'adresse du endpoint devant pointer vers le Service Bus.

```xml
<service name="SOR.Business.ServiceImpl">
  <endpoint address="https://sor.servicebus.windows.net/Service/"
            binding="ws2007HttpRelayBinding"
            bindingConfiguration="httpRelayEndpointConfig"
            behaviorConfiguration="sharedSecretClientCredentials"
            contract="SOR.Contract.IService" />
</service>
```

Ce endpoint utilise le behavior et le binding précédemment créés.

Par mesure de sécurité, le Service Bus ne permet d'exposer ni le wsdl d'un service SOAP ni un endpoint de type `IMetadataExchange` (il est en effet préférable de ne pas exposer ces informations dans un environnement de production).


## Configurer un service WCF DataServices

La configuration d'un service WCF DataServices (OData) est très légèrement différente du service WCF SOAP.

Il faut, comme précédemment, rajouter les trois extensions suivantes:

* TransportClientEndpointBehavior : Idem que pour le service WCF SOAP
* ServiceRegistrySettings : Idem que pour le service WCF SOAP
* WebHttpRelayBinding : Représente un binding de type WebHttpBinding (REST) supportant une liaison avec le Service Bus.

```xml
<extensions>
  <behaviorExtensions>
    <add name="transportClientEndpointBehavior" 
         type="Microsoft.ServiceBus.Configuration.TransportClientEndpointBehaviorElement, 
         Microsoft.ServiceBus, Version=1.6.0.0, Culture=neutral, 
         PublicKeyToken=31bf3856ad364e35"/>
    <add name="serviceRegistrySettings" 
         type="Microsoft.ServiceBus.Configuration.ServiceRegistrySettingsElement, 
         Microsoft.ServiceBus, Version=1.6.0.0, Culture=neutral, 
         PublicKeyToken=31bf3856ad364e35"/>
  </behaviorExtensions>
  <bindingExtensions>
    <add name="webHttpRelayBinding" 
         type="Microsoft.ServiceBus.Configuration.WebHttpRelayBindingCollectionElement, 
         Microsoft.ServiceBus, Version=1.6.0.0, Culture=neutral, 
         PublicKeyToken=31bf3856ad364e35"/>
  </bindingExtensions>
</extensions> 
```

Rajoutez le behavior suivant (identique au précédent à la différence prêt qu'on spécifie qu'il s'agit d'un service REST).

```xml
<endpointBehaviors>
  <behavior name="sharedSecretClientCredentials">
    <webHttp />
    <serviceRegistrySettings discoveryMode="Public">
    </serviceRegistrySettings>
    <transportClientEndpointBehavior>
      <tokenProvider>
        <sharedSecret issuerName="owner" 
                      issuerSecret="**********" />
      </tokenProvider>
    </transportClientEndpointBehavior>
  </behavior>
</endpointBehaviors>
```

Rajoutez un binding de type `WebHttpRelayBinding` (identique au binding `WS2007HttpRelayBinding` précédemment créé).

```xml
<webHttpRelayBinding>
  <binding name="webHttpRelayBindingConfiguration">
    <security relayClientAuthenticationType="None"></security>
  </binding>
</webHttpRelayBinding>
```

De la même manière que pour le service WCF SOAP, on va maintenant rajouter un endpoint à notre service, l'adresse pointant vers le Service Bus.

```xml
<service name="SOR.Business.DataService">
  <endpoint address="https://sor.servicebus.windows.net/DataService/"
            binding="webHttpRelayBinding" 
            bindingConfiguration="webHttpRelayBindingConfiguration"  
            behaviorConfiguration="sharedSecretClientCredentials" 
            contract="System.Data.Services.IRequestHandler" />
</service>
```

Le endpoint utilise le binding et behavior définis précédemment. Le contrat doit être positionné sur `System.Data.Services.IRequestHandler`, tous les services WCF DataServices implémentant cette interface (il s'agit du contrat de base).

## Configuration IIS

Maintenant que les services sont prêts à être exposés sur le Service Bus, il reste à configurer IIS.

La première action à effectuer est de vérifier que l'application pool, sous laquelle tournent les deux services, s'exécute sous une identité ayant les droits d'accès à internet (ce n'est pas toujours le cas pour les comptes système).

Pour que le service se connecte au Service Bus, il faut qu'il soit démarré et que le endpoint soit monté. Par défaut, un service n'est démarré dans IIS que lorsqu'une première requête vers ce service est effectuée. Il est donc obligatoire de mettre en place un mécanisme d'Auto-Start.

Windows Server AppFabric propose le démarrage automatique des services. L'activation de cette fonctionnalité s'effectue comme ci-dessous :

![Configuration AppFabric](https://farm9.staticflickr.com/8202/8210319295_96ea01859f_o.png =204x159 "Configuration AppFabric")

![Configuration AppFabric auto-start](https://farm9.staticflickr.com/8477/8211407894_384c64abac_o.png =417x159 "Configuration AppFabric auto-start")

Il est ensuite nécessaire d'activer le protocole `net.pipe` pour nos services. Ce protocole permet de faire fonctionner le service Management Service d'AppFabric, activé par défaut. Ce service sert à gérer le démarrage à distance des services WCF.

![Enable protocols](https://farm9.staticflickr.com/8210/8211408000_8a37a49b57_o.png =325x229 "Enable protocols")

Redémarrez IIS et c'est parti, les services sont accessibles via le Service Bus (la liaison prend environ 1 minute à se faire).

Pour vérifier le bon fonctionnement des services, vous pouvez vous connecter au service bus via un navigateur (si vous avez configuré votre service en visibilité publique).

!["Publicly Listed Services"](https://farm9.staticflickr.com/8063/8210319371_6dc6643a78_o.png =341x239 "Publicly Listed Services")

Si le service apparait, la configuration est correcte. Dans le cas contraire, pensez à vous référer à l'Event Viewer pour diagnostiquer les causes de l'erreur.

A bientôt pour d'autres aventures sur le Service Bus !

