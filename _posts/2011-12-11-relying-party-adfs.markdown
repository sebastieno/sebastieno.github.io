---
layout: post
title: 'Enregistrer une application dans ADFS'
date: 2011-12-11
categories: federation-didentite
resume: 'Comment enregistrer une application dans ADFS dans le but de déléguer son authentification à un STS (Security Token Service) de confiance ?'
tags: relying-party adfs
---
Pour que le scénario de fédération d’identité soit valide de bout en bout, il faut que l’application ait déclaré (via le fichier de configuration) que le STS (_Security Token Service_), à qui est délégué la gestion de l’authentification et de l’identité, est un STS de confiance (c’est à dire que l’application “croît” les informations renvoyées par ce STS), et il faut que le STS ait enregistré l’application comme _Relying Party_ (c’est à dire que le STS accepte d’authentifier et de renvoyer les informations d’identité de l’utilisateur à l’application).

Ce double trust est essentiel puisqu’il permet au STS de ne pas gérer l’authentification d’applications non enregistrées (et donc de ne pas renvoyer de données d’identité à n’importe quelle application) et il permet à l’application de ne pas accepter les informations d’identité émit par n’importe quel STS.

_Ce post explique comment enregistrer une application dans ADFS. Pour avoir les informations sur l’enregistrement d’une application dans ACS d’Azure, c’est ici : <a href="https://sebastienollivier.fr/blog/federation-didentite/relying-party-acs">Enregistrer une application dans ACS</a>._

## Inscription d'une Relying Party

Pour démarrer l'inscription, lancez _AD FS 2 Management_. Sélectionnez le nœud _Relying Party Trusts_ et cliquez sur l'action _Add Relying Party_.

![Relying Party Trusts](https://farm9.staticflickr.com/8058/8211353990_225dcb3e3b_o.png =221x235 "Relying Party Trusts")

Deux choix s'offrent à nous: enregistrer manuellement l'application ou utiliser le fichier FederationMetadata.xml généré par l'utilitaire _Windows Identity Foundation Federation Utility_ (utiliser le fichier permet d’enregistrer automatiquement l’application à partir des informations saisies depuis l’utilitaire).

En choisissant l'enregistrement manuel, le premier écran de configuration demande un nom pour la _Relying Party_.

![Relying Party](https://farm9.staticflickr.com/8070/8211354258_ae026e58f9_o.png =533x221 "Relying Party")

Le second écran permet de choisir si l'on souhaite utiliser le protocole SAML 2.0 ou SAML 1 / 1.1.

![ADFS Profile](https://farm9.staticflickr.com/8208/8211354222_744c33b85a_o.png =493x163 "ADFS Profile")

Il faut ensuite sélectionner un certificat si l'on souhaite encrypter le token renvoyé par ADFS.

![Certificat](https://farm9.staticflickr.com/8069/8211354206_6ccbd2b2e3_o.png =521x182 "Certificat")

L'écran suivant permet d'activer la fédération passive (par redirection vers le STS) et le SSO (Single-Sign-On). Pour faire de la fédération passive avec une application ASP.NET, il faut au minimum activer le protocole _WS-Federation Passive_ et renseigner l'URL de l'application. Sans cette activation, <span>la redirection qu'effectuera l'application vers l'ADFS sera refusée et un message d’erreur sera renvoyé</span>.

![Passive protocol URL](https://farm9.staticflickr.com/8069/8210265401_8580cd093b_o.png =531x331 "Passive protocol URL")

L'écran suivant demande un nom unique pour la _Relying Party_ (on peut laisser l'URL de l'application comme identifiant).

![Identifier](https://farm9.staticflickr.com/8339/8211354146_67e9af6587_o.png =534x254 "Identifier")

Le dernier écran, qui est disponible si on a utilisé l’enregistrement automatique, permet de configurer les autorisations par défaut des utilisateurs.

![Relying Party permissions](https://farm9.staticflickr.com/8478/8210265335_80cb759a9a_o.png =534x254 "Relying Party permissions")

La _Relying Party_ est configurée.

![Relying Party Trusts](https://farm9.staticflickr.com/8480/8210265167_9acb0ccaa1_o.png =576x66 "Relying Party Trusts")

L'application peut maintenant effectuer la redirection vers l'ADFS pour demander une authentification de l'utilisateur.

## Configuration des claims

Par défaut, ADFS ne va renvoyer aucun claim à une nouvelle _Relying Party_.

Pour configurer les claims renvoyés à l'application, sélectionnez la _Relying Party_ puis cliquez sur _Edit Claims Rules…_.

![Edit Claims Rules](https://farm9.staticflickr.com/8340/8210265253_72b8177f5c_o.png =357x384 "Edit Claims Rules")

Sur l'écran d'édition des claims, vous aurez la possibilité d'ajouter/modifier et supprimer des règles.

Il y a 5 types de règles différentes :

* Send LDAP Attributes as Claims : renvoyer des attributs LDAP en tant que Claims
* Send Group Membership as a Claim : renvoyer un Claim en fonction de l'appartenance de l'utilisateur à un groupe AD
* Transform an Incoming Claim : transformer un Claim envoyé à l'ADFS (par les _Identity Providers_) et le renvoyer
* Pass Through or Filter an Incoming Claim : Renvoyer ou filtrer les claims envoyés à l'ADFS (par les _Identity Providers_)
* Send Claims Using a Custom Rule : Utiliser un _Custom Attribute Store_ pour renvoyer des Claims (un _Custom Attribute Store _permet de récupérer des Claims à partir d’une source externe à l’ADFS, comme un web service métier)

## Récupération des claims côté applicatif

Depuis l'application .NET, la liste des Claims est accessible depuis la propriété `Claims` de la classe `ClaimsIdentity`.

![ClaimsIdentity](https://farm9.staticflickr.com/8197/8210265295_39dd78e197_o.png =1368x276 "Claims Identity")

A bientôt !


