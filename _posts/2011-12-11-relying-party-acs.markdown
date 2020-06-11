---
layout: post
title: 'Enregistrer une application dans ACS'
date: 2011-12-11
categories: federation-didentite
resume: 'Comment enregistrer une application dans ACS dans le but de déléguer son authentification à un STS (Security Token Service) de confiance ?'
tags: relying-party acs
---
Pour que le scénario de fédération d’identité soit valide de bout en bout, il faut que l’application ait déclaré (via le fichier de configuration) que le STS (_Security Token Service_ ), à qui est délégué la gestion de l’authentification et de l’identité, est un STS de confiance (c’est à dire que l’application “croît” les informations renvoyées par ce STS), et il faut que le STS ait enregistré l’application comme _Relying Party_ (c’est à dire que le STS accepte d’authentifier et de renvoyer les informations d’identité de l’utilisateur à l’application).

Ce double trust est essentiel puisqu’il permet au STS de ne pas gérer l’authentification d’applications non enregistrées (et donc de ne pas renvoyer de données d’identité à n’importe quelle application) et il permet à l’application de ne pas accepter les informations d’identité émit par n’importe quel STS.

_Ce post explique comment enregistrer une application dans ACS d’Azure. Pour avoir les informations sur l’enregistrement d’une application dans ADFS, c’est ici : <a href="https://sebastienollivier.fr/blog/federation-didentite/relying-party-adfs">Enregistrer une application dans ADFS</a>._

## Inscription d'une Relying Party

Pour démarrer l'inscription, connectez-vous à Access Control Service (l’URL est sous la forme https://&lt;nom_acs&gt;.accesscontrol.windows.net/) puis cliquez sur _Relying Party applications._

![Relying party applications](https://farm9.staticflickr.com/8067/8210294525_f711d67bb7_o.png =134x221 "Relying party applications")

Cliquez sur _Add_ pour ajouter une _Relying Party_.

Après avoir renseigné le nom de la _Relying Party_, deux choix s'offrent à nous: enregistrer manuellement l'application ou utiliser le fichier FederationMetadata.xml généré par l'utilitaire _Windows Identity Foundation Federation Utility_ (utiliser le fichier permet d’enregistrer automatiquement l’application à partir des informations saisies depuis l’utilitaire).

![Relying Party Application Settings](https://farm9.staticflickr.com/8347/8210294489_788e16d654_o.png =281x156 "Relying Party Application Settings")

En choisissant l'enregistrement manuel, il faut renseigner l’URL de l’application, l’URL de retour (utilisée par ACS pour envoyer le token à l’application) et une URL optionnelle de retour en cas d’erreur.

![URLs configuration](https://farm9.staticflickr.com/8344/8211382838_72f487170f_o.png =414x206 "URLs configuration")

On a ensuite la possibilité de choisir entre le protocole SAML 2.0, SAML 1.1 ou SWT, de choisir si l’on souhaite encrypter le token renvoyé par ACS (uniquement si on a choisi l’option configuration manuelle) et de choisir la durée de validité du token.

![Token configuration](https://farm9.staticflickr.com/8066/8210294375_2327c970cb_o.png =486x171 "Token configuration")

La deuxième partie de l’écran permet de configurer l’authentification. On doit sélectionner les _Identity Providers_ que l’on souhaite utiliser pour l’application. On a ensuite le choix entre créer un nouveau groupe de règles de génération de Claims ou réutiliser un groupe existant.

![Providers configuration](https://farm9.staticflickr.com/8067/8210294347_2c447d2e41_o.png =431x153 "Providers configuration")

La dernière partie de l’écran permet de spécifier le certificat avec lequel le token sera signé.

![Signing configuration](https://farm9.staticflickr.com/8199/8210294287_06c089a0dd_o.png =387x164 "Signing configuration")

## Configuration des claims

Par défaut, ACS va créer un nouveau groupe de règles vide pour la _Relying Party_. Ce groupe va nous permettre de configurer les claims renvoyées par ACS à l’application et pourra être réutilisé par d’autres _Relying Party_.

Cliquez sur _Rule groups_ sur le menu de gauche puis sélectionner le groupe de règle associé à notre _Relying Party_ (si vous avez choisi l’option _Create new rule group_, un nouveau groupe portant le nom de la _Relying Party_ sera créé).

![Rule groups](https://farm9.staticflickr.com/8341/8211382594_bcc66d36c9_o.png =200x150 "Rule groups")

ACS propose, via le bouton _Generate_, de générer des règles pré-définies pour les différents _Identity Providers_ disponibles.

![Generate rules](https://farm9.staticflickr.com/8478/8210293971_9fe83b394e_o.png =437x110 "Generate rules")

Il est possible d’ajouter manuellement une règle en cliquant sur _Add_. La première partie d’écran permet de définir quel claim envoyé à ACS de quel _Identity Provider_ va être concerné par la règle (la configuration ci-dessous permet de créer une règle sur les claims de type `nameidentifier` envoyé par Windows Live ID).

![If](https://farm9.staticflickr.com/8490/8210294079_40bfab08cf_o.png =391x299 "If")

La deuxième partir de l’écran permet de définir les transformations à effectuer sur le claim envoyé à ACS et quel claim sera renvoyé par ACS (la configuration ci-dessous permet de renvoyer à la _Relying Party_ un claim de type `name` contenant la valeur du claim de type `nameidentifier` envoyé par Windows Live ID).

![Then](https://farm9.staticflickr.com/8198/8211382502_f36da7ffdf_o.png =451x187 "Then")

## Récupération des claims côté applicatif

Depuis l'application .NET, la liste des Claims est accessible depuis la propriété `Claims` de la classe `ClaimsIdentity`.

![Claims](https://farm9.staticflickr.com/8061/8210294217_3892c5d378_o.png =1158x274 "Claims")

A bientôt !

