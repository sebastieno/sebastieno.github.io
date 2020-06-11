---
layout: post
title: 'Envoyer des notifications Push sur un site Web'
date: 2018-01-24
categories: web
resume: 'La Push API est une spécification W3C décrivant la façon dont un service de notifications peut envoyer un message Push à une application Web, via le protocol Web Push. Voyons ensemble comment ça marche.'
tags: pwa service-worker push-api webpush notification
---
La Push API est une <a title="W3C Push Api spécification" href="https://www.w3.org/TR/push-api/" target="_blank">spécification W3C</a> décrivant la façon dont un service de notifications peut envoyer un message Push à une application Web, via le protocole <a title="Web Push" href="https://www.w3.org/TR/push-api/#dfn-web-push-protocol" target="_blank">Web Push</a>. Elle décrit également le processus à mettre en place côté client pour établir la communication entre le navigateur et le service de notifications.

_Cette spécification est en cours de rédaction (Working Draft), c'est-à-dire qu'elle est assez avancée pour pouvoir être implémentée par les navigateurs mais qu'elle peut être amenée à changer légèrement en fonction des retours._

A l'heure où j'écris ces lignes, le support de la Push API n'est pas encore global à l'ensemble des navigateurs mais commence à être assez complet pour pouvoir envisager des scénarios réels :

![Push API Support](https://sebastienollivier.blob.core.windows.net/blog/envoyer-des-notifications-push-sur-un-site-web/pushapi-support.png =1261x457 "Push API Support")

Voyons ensemble comment ça marche.

## Vérifier si le navigateur supporte les notifications

Comme vu précédemment, le support de la Push API n'est pas global. Il convient donc de vérifier si le navigateur supporte cette spécification avant de continuer :

```javascript
if ('serviceWorker' in navigator && 'PushManager' in window) { 
    // It's ok
}
```

Si l'objet `PushManager` est présent sur `window`, la spécification est supportée par le navigateur. Il est également nécessaire de tester si les <a title="Service Worker : Rendre votre site accessible même sans connexion !" href="https://blogs.infinitesquare.com/posts/web/service-worker-rendre-votre-site-accessible-meme-sans-connexion" target="_blank">serviceWorker</a> sont supportés, puisqu'ils seront nécessaires c&ocirc;té client pour récupérer les notifications.

## Demander le consentement à l'utilisateur

Avant de pouvoir établir la communication entre le serveur de notifications et le navigateur, il faut demander à l'utilisateur s'il accepte d'être notifié. Cela se fait en appelant la méthode `requestPermission` de l'objet `Notification` :

```javascript
Notification.requestPermission().then(function(result) {});
```

La promise renvoyée par la méthode contient le résultat de la demande. Ce résultat peut avoir trois valeurs : `granted` (permission accordée), `denied` (permission refusée) ou `default` (décision inconnue, à considérer comme un refus).

![Browser permission](https://sebastienollivier.blob.core.windows.net/blog/envoyer-des-notifications-push-sur-un-site-web/browser-pushpermission.png =329x159 "Browser permission")

La propriété `permission` de l'objet `Notification` permet de récupérer la permission sans avoir à effectuer la demande (sa valeur sera à `default` si la demande n'a jamais été effectuée) :

![Notification.permission property](https://sebastienollivier.blob.core.windows.net/blog/envoyer-des-notifications-push-sur-un-site-web/notification-permission-property.png =204x48 "Notification.permission property")

_Attention, demander la permission directement lors de l'accès à l'application peut être troublant pour l'utilisateur. Il ne connait pas forcément l'intention derrière cette demande d'autorisation de notifications, ce qui peut être effrayant. Voici quelques conseils afin d'améliorer l'UX : <a title="Permission UX" href="https://developers.google.com/web/fundamentals/push-notifications/permission-ux" target="_blank">https://developers.google.com/web/fundamentals/push-notifications/permission-ux</a>._

## Souscrire aux notifications

Si l'utilisateur nous donne l'autorisation, nous allons pouvoir créer la souscription aux notifications. Cette souscription permet d'accéder aux informations nécessaires à l'envoi de notifications à l'utilisateur (endpoint, clef de sécurité, etc.).

Pour cela, nous devons utiliser la méthode `subscribe` de la propriété `pushManager` de l'objet `registration` renvoyé par l'inscription du service worker (pour plus d'informations sur les services workers, je vous conseille la lecture de l'article suivant <a title="Service Worker : Rendre votre site accessible même sans connexion !" href="https://blogs.infinitesquare.com/posts/web/service-worker-rendre-votre-site-accessible-meme-sans-connexion" target="_blank">Service Worker : Rendre votre site accessible même sans connexion !</a>) :

```javascript
Notification.requestPermission().then(function (result) {    
    if (result === 'granted') {
        navigator.serviceWorker.register('sw.js');
        navigator.serviceWorker.ready.then(function (registration) { 
            registration.pushManager.subscribe([...]);
        })
    }
});
```

Cette méthode attend en paramètre un objet contenant deux propriétés :

* userVisibleOnly : indique si les notifications seront systématiquement affichées à l'utilisateur ou s'il y aura des notifications "silencieuses". _Attention, certains navigateurs (comme Chrome) ne supportent pas les notifications "silencieuses", qui posent d'évidents problèmes de confidentialité puisque l'utilisateur n'est pas notifié de la communication avec le serveur._
* applicationServerKey : Clef publique utilisée pour la notification

_La sécurisation de la communication entre le serveur de push et le navigateur est effectuée gr&acirc;ce à un système de couple clef privée clef publique. L'idée est de créer la souscription en fournissant la clef publique (cette clef n'ayant pas besoin de rester confidentielle) puis d'envoyer les notifications en utilisant la clef privée (clef devant absolument être gardée confidentielle). De cette manière, on s'assure qu'un tiers ne puisse pas envoyer des notifications sur votre application Web. Vous pouvez utiliser le site <a href="https://web-push-codelab.glitch.me/,">https://web-push-codelab.glitch.me</a> pour générer un couple de clef public / privée._

```javascript
Notification.requestPermission().then(function (result) {
    if (result === 'granted') {
        navigator.serviceWorker.register('sw.js');
        navigator.serviceWorker.ready.then(function (registration) {
            registration.pushManager.subscribe({
                userVisibleOnly: true,
                applicationServerKey: urlB64ToUint8Array('<Remplacez par votre clef publique>')
            }).then(function (pushSubscription) {
                console.info('Subscription informations', JSON.stringify(pushSubscription));
            });
        })
    }
});
```

A la résolution de la promise, toutes les informations nécessaires à l'envoi de notifications push seront disponibles. L'objet renvoyé prend la forme du json suivant :

```javascript
{
    "endpoint":"https://fcm.googleapis.com/fcm/send/fBkqJAUP4vo:APA91bGJpu8dF...",
    "expirationTime":null,
    "keys":{
        "p256dh":"BE84qCK95nvgYttlxjxXOR8KdHDrZ08IvWiOY0JQedGUNP8s9qJdACfIV0qjKTVGNJLnhH39OQPhjAblebxPLQU=",
        "auth":"q7NXuI9PkfvJkOuaMKSPig=="
    }
}
```
## Envoi des informations de souscription au serveur
Une fois ces informations récupérées, il va falloir les envoyer au serveur application (API). De cette manière, on pourra les stocker en les associant à l'utilisateur pour ensuite les récupérer lorsque l'on souhaitera envoyer une notification.

Cette partie est donc spécifique à votre environnement, donc je vous laisse vous débrouiller :).

## Envoi d'une notification Push

L'envoi d'une notification Push se fait en utilisant les informations de souscription. Chaque navigateur va fournir les informations nécessaires à l'envoi de push, notamment un endpoint respectant le protocole <a title="Web Push" href="https://www.w3.org/TR/push-api/#dfn-web-push-protocol" target="_blank">Web Push</a>. L'idée est donc d'effectuer une requête POST sur le endpoint fourni, en ayant au préalable positionné les headers d'Authorization. Pour nous faciliter la t&acirc;che, un ensemble de librairies existe : <a href="https://github.com/web-push-libs/">https://github.com/web-push-libs/</a>.

Voici un exemple d'envoi de notification Push via la librairie <a title="Web Push library" href="https://github.com/web-push-libs/web-push" target="_blank">web-push</a> pour Node.JS

```javascript
const webpush = require('web-push');
webpush.setVapidDetails('mailto:sebastien.ollivier@gmail.com', 'BH4YZ_yQDkf77ZGs361qyO24CNswEFDrd4zcTJMVTqqr1kgAC6t8eTrPMTCnZfXcOuoyuKMvRLT-XQa9E7ld_sk', 'eh-xZQ14_AXjDpgBci1Hm3x3HQIOGCCI4mL9Aa13qcY');

const pushSubscription = {
    endpoint: "https://fcm.googleapis.com/fcm/send/fBkqJAUP4vo:APA91bGJpu8dFJu5sHTLRZUvzXTnFrLyjNzkr6OJAcDKuPlbJ4YK6z...",
    keys: {
        auth: "q7NXuI9PkfvJkOuaMKSPig==",
        p256dh: "BE84qCK95nvgYttlxjxXOR8KdHDrZ08IvWiOY0JQedGUNP8s9qJdACfIV0qjKTVGNJLnhH39OQPhjAblebxPLQU="
    }
};

webpush.sendNotification(pushSubscription, 'Your Push Payload Text');
```

La méthode `setVapidDetails` de l'objet `webpush` permet de renseigner le couple clef privée / clef publique (ce code s'exécute côté serveur donc pas de problème de confidentialité concernant la clef privée). Ensuite, la méthode `sendNotification` permet d'envoyer la notification, en spécifiant les informations de souscription récupérées depuis l'étape _Souscrire aux notifications_.

La notification est envoyée, il ne reste plus qu'à la réceptionner et à l'afficher.

## Réception et affichage de la notification Push

La réception des notifications se fait en s'abonnant à l'évent `push` depuis le <a title="Service Worker : Rendre votre site accessible même sans connexion !" href="https://blogs.infinitesquare.com/posts/web/service-worker-rendre-votre-site-accessible-meme-sans-connexion" target="_blank">serviceWorker</a> (c'est pour cela qu'il est nécessaire de vérifier que le navigateur supporte cette spécification, et que votre application soit exposée via le protocole HTTPS sauf pour localhost) :

```javascript
self.addEventListener('push', function(event) {
});
```

L'affichage d'une notification se fait en appelant la méthode _showNotification_ de l'object _registration_ : 

```javascript
self.addEventListener('push', function(event) {
    const promise = self.registration.showNotification('Notification title', {
        body: event.data.text()
    });

    event.waitUntil(promise);
});
```

Cette méthode renvoie une promise et prend en paramètre un titre et des options, permettant d'indiquer une icône, un contenu, etc. (pour plus d'informations sur les options, vous pouvez vous référer à la documentation MDN : <a href="https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerRegistration/showNotification">https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerRegistration/showNotification</a>).

Ce qui est intéressant de noter dans le code précédent est la ligne `event.waitUntil(notificationPromise)`. Cette ligne permet d'étendre la durée de vie de l'évènement de notification de manière à indiquer au navigateur d'attendre sa résolution (donc l'affichage de la notification) avant de désallouer les ressources du service worker (ce qui aurait pour conséquence d'empêcher l'affichage de la notification si elle ne s'affichage pas assez vite).

Et voilà, nous pouvons maintenant envoyer des notifications à nos utilisateurs Web ! Attention par contre à ne pas abuser des notifications au risque de les poluer.

Tout le code source de cet article est disponible sur mon GitHub à l'adresse suivante : <a title="https://github.com/sebastieno/webpush-demo" href="https://github.com/sebastieno/webpush-demo" target="_blank">https://github.com/sebastieno/webpush-demo</a>.

Bonnes notifications !

