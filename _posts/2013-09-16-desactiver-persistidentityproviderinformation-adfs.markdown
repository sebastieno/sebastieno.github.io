---
layout: post
title: 'Désactiver la sauvegarde de l''IdP sélectionné sur ADFS'
date: 2013-09-16
categories: federation-didentite
resume: 'Comment désactiver la sauvegarde du provider d''identité sur ADFS afin que sa sélection soit nécessaire à chaque connexion ?'
tags: disable-selected-idp adfs
---
Quand un utilisateur s'authentifie sur un site utilisant ADFS, il doit sélectionner le provider d'identité (IdP) qu'il souhaite utiliser (AD, LiveID, etc.), si ADFS est configuré pour utiliser plusieurs IdP. Pour que l'utilisateur n'ait pas à sélectionner à chaque connexion l'IdP, ADFS va sauvegarder  ce choix dans un cookie. Pour pouvoir de nouveau sélectionner un IdP, l'utilisateur devra attendre l'expiration du cookie ou le supprimer, utiliser un autre navigateur ou utiliser une navigation InPrivate.

Dans certains scénarios, on va avoir besoin de désactiver cette sauvegarde pour que la sélection d'un IdP soit nécessaire à chaque connexion, notamment si un ordinateur est utilisé par plusieurs utilisateurs.

Généralement, lorsque l'on souhaite modifier le comportement d'ADFS, il est nécessaire de faire un développement custom (par exemple si l'on souhaite ajouter une sélection automatique de l'IdP en fonction de l'email de l'utilisateur).

Dans notre cas, on va pouvoir désactiver la sauvegarde de l'IdP simplement en modifiant le fichier de configuration de ADFS, situé dans le répertoire C:\inetpub\adfs\ls\. Le noeud concerné est le noeud `persistIdentityProviderInformation`, l'attribut `enabled` permettant d'activer la sauvegarde de l'IdP et l'attribut `lifetimeInDays` permettant de spécifier combien de temps l'IdP sélectionné par l'utilisateur sera sauvegardé.

Par défaut, la configuration est la suivante (sauvegarde activé pour 90 jours) :

```xml
<persistIdentityProviderInformation enabled="true" lifetimeInDays="90" />
```

Pour désactiver la sauvegarde, il faut utiliser la configuration suivante :

```xml
<persistIdentityProviderInformation enabled="false" />
```

Bonne configuration !


