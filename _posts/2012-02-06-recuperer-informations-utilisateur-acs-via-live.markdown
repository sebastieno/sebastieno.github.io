---
layout: post
title: 'Récupérer les informations de l''utilisateur authentifié depuis ACS via Live ID'
date: 2012-02-06
categories: federation-didentite
resume: 'Utilisons le SDK Live pour récupérer les informations d''un utilisateur authentifié avec son compte Live ID depuis ACS.'
tags: acs live claims
---
Lorsqu'on utilise Live ID comme Identity Provider sur ACS, le seul Claim renvoyé par Live ID est un hash de l'UID du compte Live utilisé. Les informations telles que l'adresse email, le nom ou le prénom ne sont pas renvoyées.

Pour retrouver ces informations, la plupart des sites internet utilisant Live ID propose un formulaire d'enregistrement où l'utilisateur authentifié pourra renseigner ses informations (au moins son email). Au delà du fait qu'on oblige l'utilisateur à renseigner des informations qu'il a déjà renseignées à la création de son compte Live, cette solution n'est pas fiable puisqu'on ne peut pas vérifier que l'email renseigné correspond à l'email avec lequel l'utilisateur s'est logué, à moins d'envoyer un email à cette adresse contenant une URL de validation d'inscription… mais là le processus d'inscription devient infernal pour l'utilisateur. 

Le SDK Live permet d'accéder aux informations du compte Live ID de l'utilisateur logué, en utilisant le même principe de consentement que Facebook et GMail.

Le principe est simple. Notre application va être enregistrée auprès de Live ID. Lorsqu'on va demander les informations utilisateur, le SDK Live va vérifier les droits de notre application sur le compte Live ID. Si aucun droit n'existe, le SDK va demander à l'utilisateur s'il accepte d'autoriser l'accès aux informations de son compte (emails, contacts, calendriers, etc. selon ce que l'application a demandé). En fonction du choix de l'utilisateur, on pourra accéder ou non à ses informations.

## Enregistrement de l'application

Il faut dans un premier temps enregistrer l'application sur <a href="https://manage.dev.live.com/" target="_blank">https://manage.dev.live.com/</a> en cliquant sur _Create Application_. Il est nécessaire de renseigner un nom d'application (utilisé pour indiquer à l'utilisateur quelle est l'application qui souhaite accéder à ses informations) et de choisir une langue (utilisée pour afficher un message localisé à l'utilisateur).

![Application name](https://farm9.staticflickr.com/8489/8211406630_cb4705c5c2_o.png =453x148 "Application name")

L'application est alors référencée par Live ID.

![Application informations](https://farm9.staticflickr.com/8480/8210317915_d1d241bfde_o.png =467x104 "Application informations")

Dans l'onglet _API Settings_ de la page de configuration, on renseigne le domaine qui sera utilisé par Live ID pour effectuer une redirection (sans cette information, le SDK Interactive Live n'acceptera pas de renvoyer les informations).

![Application settings](https://farm9.staticflickr.com/8207/8211406682_ec8cd70b43_o.png =642x201 "Application settings")

Une fois l'enregistrement terminé, on va pouvoir depuis l'application envoyer une requête au SDK Live 
pour récupérer les informations nécessaires.

Il est possible de requêter ces informations de deux manières différentes: JavaScript ou REST.

## Accès aux informations en JavaScript

Le SDK Live met à disposition un fichier JS contenant un ensemble de fonctions permettant d'interagir en JavaScript avec l'API.

```js
<script type="text/javascript" src="https://js.live.net/v5.0/wl.js" ></script>
```

L'objet WL va nous permettre de récupérer les informations utilisateur. La première étape consiste à initialiser le contexte en fournissant le client Id de l'application (disponible au moment de l'enregistrement) ainsi que l'URL de l'application (aucune redirection ne sera effectuée).

```js
WL.init({ client_id: ‘000000004807A356',
        redirect_uri: 'https://acsdemo.contoso.com/MvcApplication' });
```

Le contexte étant initialisé, on va pouvoir authentifier l'utilisateur auprès du SDK en spécifiant un scope. Ce scope correspond aux données que l'on souhaite récupérer dans l'application (email, contacts, calendriers, etc.).

```js
WL.login({ "scope": "wl.basic" },
         function (response)
         {
             if (response.status == "connected")
             { 
                // utilisateur connecté
             }
         });
```

Etant donné que l'utilisateur aura déjà fourni ses credentials Live ID via ACS, il ne sera pas redirigé vers la page d'authentification Live.

A la place de cette page, une fenêtre de consentement va être affichée demandant à l'utilisateur s'il accepte de fournir ses informations (en fonction du scope demandé) à notre application. Si la demande de consentement a déjà été validée (lors d'une visite précédente), une fenêtre va demander à l'utilisateur de retaper son mot de passe.

![Demande de consentement](https://farm9.staticflickr.com/8344/8210317963_84aab38a7f_o.png =403x312 "Demande de consentement")

<p style="text-align: center;"><em>Première connexion au site : Demande de consentement</em></p>

![Resaisie du mot de passe](https://farm9.staticflickr.com/8065/8210318015_15c4995aa1_o.png =318x 301 "Resaisie du mot de passe")

<p style="text-align: center;"><em>Connexions suivantes (le consentement a déjà été effectué) : Retaper son mot de passe</em></p>

L'utilisateur est maintenant authentifié et a autorisé notre application à accéder à des informations.

Il reste à demander au SDK de nous renvoyer les informations.

```js
WL.api("/me", "GET",
       function (response)
       {
           // récupération des données effectuées
       });
```

Les informations au format JSON sont disponibles via la variable response. Ci-dessous un exemple de réponse :

```js
{
    "id": "xxxxxx",
    "name": "Sebastien Ollivier",
    "first_name": "Sebastien",
    "last_name": "Ollivier",
    "link": "http://profile.live.com/cid-xxxx/",
    "gender": null,
    "emails": {
        "preferred": "sebastien.ollivier@hotmail.com",
        "account": "sebastien.ollivier@hotmail.com",
        "personal": null,
        "business": null
    },
    "locale": "fr_FR",
    "updated_time": "2012-01-24T16:03:02+0000"
}

```

## Accès aux informations en REST

La deuxième manière de requêter les informations utilisateur est d'utiliser REST (contrairement au JavaScript, il y aura nécessairement une redirection).

Avant de pouvoir demander les informations utilisateur, il va falloir récupérer un access token valide. Pour cela, on va utiliser Live Connect et effectuer une redirection vers l'URL suivante : _https://oauth.live.com/authorize?client_id=xxx&scope=wl.basic&response&#95;type=token&redirect&#95;uri=xxx_

De la même manière qu'en JavaScript, il faut renseigner le client Id de l'application, le scope souhaité ainsi que l'URL de l'application (utilisée pour effectuer la redirection).

Après que l'utilisateur ait validé la demande de consentement ou retapé son mot de passe, une redirection sera effectuée vers notre application en fournissant le token dans l'URL, de la manière suivante: _https://acsdemo.contoso.com/MvcApplication/LiveId/#access&#95;token=EwAwAq1DB[…]_

Ce token doit ensuite être utilisé pour appeler l'API Live de la manière suivante :
_https://apis.live.net/v5.0/me?access&#95;token=&lt;access&#95;token&gt;_

Les informations sont renvoyées au format JSON, comme via le requêtage JavaScript.

Pour plus d'informations sur le SDK Interactive Live, <a href="http://isdk.dev.live.com/">http://isdk.dev.live.com/</a>.

La liste des droits des applications liés à un compte est disponible sur <a href="https://consent.live.com/">https://consent.live.com/</a>.

![Liste des droits des applications](https://farm9.staticflickr.com/8208/8210318053_f6594d0ace_o.png =482x147 "Liste des droits des applications")

## Claims des Identity Providers

Pour information, voici les claims renvoyés par les principaux Identity Providers:

### LiveID

http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier


### Google

http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier


http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress


http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name


### Yahoo

http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier


http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress


http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name


### Facebook

http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier


http://schemas.microsoft.com/ws/2008/06/identity/claims/expiration


http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress


http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name


http://www.facebook.com/claims/AccessToken

<br>

En plus de ces claims, ACS renvoie un claim représentant le provider utilisé : 

http://schemas.microsoft.com/accesscontrolservice/2010/07/claims/identityprovider


