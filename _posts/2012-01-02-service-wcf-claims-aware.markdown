---
layout: post
title: 'Service WCF Claims Aware'
date: 2012-01-02
categories: federation-didentite
resume: 'Comment sécuriser un service WCF via délégation d''authentification à un STS en utiliser le SDK Windows Identity Foundation (WIF) ?'
tags: wcf claims
---
De la même manière que pour une application ASP.NET, on peut utiliser la fédération d’identité pour sécuriser un service WCF.

On va voir dans ce post comment rendre un service WCF claims-aware, via l’utilitaire Windows Identity Foundation Federation Utility.

## Windows Identity Foundation Federation Utility

Pour lancer l'utilitaire, cliquez droit, dans Visual Studio, sur _Add STS reference…_ à partir d'un projet WCF (si l'utilitaire ne se lance pas, vous pouvez l'exécuter depuis le menu _Administration Tools_ de Windows).

![Add STS reference...](https://farm9.staticflickr.com/8057/8210322039_6e298690d6_o.png =364x108 "Add STS reference...")

Le premier écran présente les informations sur notre service. On doit y renseigner l'emplacement du fichier de configuration (qui sera modifié par l'utilitaire) et l'URL du service.

![Federation utility wizard](https://farm9.staticflickr.com/8483/8211410406_60349a638d_o.png =496x375 "Federation utility wizard")

Le deuxième écran permet de sélectionner le STS responsable de l’authentification. Pour utiliser ACS (Access Control Service, STS version Cloud de Microsoft) ou ADFS (Active Directory Federation Services, STS version On-Premise de Microsoft), on doit sélectionner l'option _Use an existing STS_ et renseigner l'adresse du fichier `FederationMetadata.xml` du STS, fichier contenant les informations de configuration.

![Federation utility wizard](https://farm9.staticflickr.com/8064/8211410624_353fafb634_o.png =508x384 "Federation utility wizard")

L’écran suivant permet d’activer le cryptage du token SAML (obligatoire pour un service WCF) en fournissant un certificat.

![Federation utility wizard](https://farm9.staticflickr.com/8059/8211410564_d58d6f46d2_o.png =512x387 "Federation utility wizard")

Après validation, l'utilitaire va modifier le fichier de configuration de notre service.

## Web.config

Une `behaviorExtension` est ajoutée. Cette extension permet de surcharger le `ServiceHost` de WCF en y ajoutant les mécanismes liés à la fédération d’identité.

![behaviorExtension](https://farm9.staticflickr.com/8338/8220098473_4a06c40388_o.png =610x80 "behaviorExtension")

Le `behavior` du service va être modifié pour y activer l’extension précédemment déclarée et pour ajouter le certificat sélectionné via l’utilitaire.

![behavior](https://farm9.staticflickr.com/8346/8220098407_ec16b4f8ed_o.png =622x186 "behavior")

Un binding `ws2007FederationHttpBinding` (basé sur `ws2007HttpBinding` et supportant la sécurité fédérée) est ajouté.

![ws2007FederationHttpBinding](https://farm9.staticflickr.com/8066/8221178878_a3891174c8_o.png =615x268 "ws2007FederationHttpBinding")

Ce binding est lié à notre service via un mapping sur le protocol HTTP.

![protocolMapping](https://farm9.staticflickr.com/8197/8221179152_04172e3953_o.png =361x45 "protocolMapping")

La section `microsoft.IdentityModel` est la section qui contient toutes les informations de configuration nécessaire au Framework WIF.

![microsoft.IdentityModel](https://farm9.staticflickr.com/8343/8221179078_1410dbe2ec_o.png =634x206 "microsoft.IdentityModel")

Le token SAML renvoyé par le STS contient une `audienceUri` correspondant à l'URL de l'application (au sens large) pour laquelle le token a été créé. La section `audienceUris` permet de spécifier la liste des URL qui seront considérées comme appartenant à notre service. Cela permet d'éviter qu'un token créé pour une application ne soit réutilisé dans notre application.

La section `issuerNameRegistry` contient la liste des STS à qui l'on accepte de déléguer la gestion de l’authentification. Cette section est très importante puisqu'elle permet d'indiquer quels sont les STS de confiance. Les tokens renvoyés par les STS enregistrés dans cette section seront considérés comme valides et l'identité de l'utilisateur se basera sur le contenu de ces tokens.

En plus de modifier le fichier de configuration, l'utilitaire va générer un fichier `FederationMetadata.xml` qui nous permettra d'inscrire notre application au STS de manière simplifiée.

## Claims Identity

Le service utilise maintenant une sécurité basée sur la fédération d’identité. Pour pouvoir appeler une méthode du service, il faudra envoyer dans les header de la requête le token créé par le STS (j’expliquerai dans des posts suivants comment acquérir ce token via fédération passive et active).

L’identité de l’utilisateur accédant au service est disponible via `Thread.CurrentPrincipal`.

![Claims](https://farm9.staticflickr.com/8199/8210322087_199e6f0a90_o.png =505x100 "Claims")

La propriété `Claims` de la classe `ClaimsIdentity` permet d'accéder à la liste des claims renvoyée par le STS.

A bientôt :)

