---
layout: post
title: 'Progressive Web App ou comment se passer des stores ?'
date: 2017-09-19
categories: web
resume: 'Progressive Web App est un terme à la mode ces derniers temps et promet une révolution dans la façon d''appréhender la notion d''applications mobiles. Faisons ensemble un tour d''horizon des PWA.'
tags: pwa service-worker web-manifest push push-api
---
Progressive Web App est un terme à la mode ces derniers temps. L'idée derrière ce terme est de créer des applications web, donc accessible depuis un navigateur, qui proposent des expériences d'utilisation identiques à ce qu'on a l'habitude de voir sur des applications natives.

L'objectif n'est pas de remplacer les applications natives, que l'on peut déjà adresser avec une application web en utilisant l'approche hybride notamment avec le Framework <a title="1 code 4 applications : Cordova, mythe ou realité ?" href="https://sebastienollivier.fr/blog/cordova/1-code-4-applications-cordova-mythe-ou-realite" target="_blank">Cordova</a>, mais de proposer des applications web avec une qualité identique tout en se passant du store.

Bien que vecteur d'une visibilité importante, déployer une application sur les stores représente de nombreuses contraintes : cycle de déploiement lent (la soumission d'une application ou d'une mise à jour peut prendre jusqu'à plus d'une semaine sur certains stores), monétisation partielle (le store prend une partie des achats inapp, de la régie publicitaire et du coût des applications si elles sont payantes), taille des applications importantes, etc.

## Mais pourquoi je voudrais me passer des stores ? C'est important pour ma visibilité !
La question que l'on pourrait alors se poser est pourquoi est-ce que je voudrais me passer des stores ? Avoir une application sur les stores (à minima iOS et Android) permet d'avoir une visibilité importante et est indispensable sur certains secteurs métiers. Aujourd'hui, cette constatation est vraie, mais plusieurs études ont démontré que l'usage d'applications stores n'est pas celui attendue.

La première statistique intéressante concerne le temps passé par application. On constate ici que sur téléphone, presque de 80% du temps d'utilisation est passé sur 3 applications (généralement application Mail, Facebook, Twitter, YouTube, etc.). A moins que vous ne travailliez pour ces entreprises, votre application devra donc se partager les 20% du temps d'utilisation restants avec toutes les autres applications.

![Individuals' top ranked app by usage](https://sebastienollivier.blob.core.windows.net/blog/progressive-web-app-ou-comment-se-passer-des-stores/stat1.png =795x396 "Individuals' top ranked app by usage")
_Source : Quartz | qz.com, Data : comScore_

La deuxième statistique intéressante concerne le nombre d'applications téléchargées par utilisateur. On y constate ici que près de la moitié des utilisateurs ne téléchargent aucune application.

![Smartphone users' number of app downloads per month](https://sebastienollivier.blob.core.windows.net/blog/progressive-web-app-ou-comment-se-passer-des-stores/stat2.png =787x375 "Smartphone users' number of app downloads per month")
_Source : Quartz | qz.com, Data : comScore_

Enfin, la dernière constatation concerne le taux d'abandon causé par un temps de téléchargement trop long : en moyenne, une visite sur deux est abandonnée au-delà de 3 secondes de chargement. Quand on sait que les applications stores sont très lourdes (119 Mo pour l'application Twitter sur iOS, 23 Mo sur Android), la légèreté des applications Web (<a title="Mobile Twitter" href="https://mobile.twitter.com" target="_blank">https://mobile.twitter.com</a> pèse moins de 1 Mo) peuvent permettre une adhésion nettement plus forte.

Bien que révélatrice d'un usage moins important que prévu, et donc d'une visibilité moins optimale, il faut modérer les conclusions que l'on peut tirer de ces statistiques avec le fait que 80% du temps d'utilisation d'un téléphone est passé sur des applications. La présence d'une application dans les stores, pour certains secteurs d'activités, est donc encore indispensable. Mais les usages évoluent et les progressive web app aideront sans doute à bousculer ce constat.

## Et alors, qu'est-ce que c'est qu'une Progressive Web App ?
Progressive Web App n'est pas un Framework. Ce n'est pas non plus un standard. Il s'agit plutôt d'un ensemble de guidelines, basés sur des API HTML, sur de l'ergonomie ou sur des bonnes pratiques, permettant à une application web de proposer une expérience d'utilisation similaire à celle d'une application native. Cela implique que l'utilisateur pourra ajouter un raccourci vers l'application web sur son téléphone (qui appara&icirc;tra comme une application native), envoyer des notifications push , avoir accès à certaines fonctionnalités du device et surtout fonctionner sans connexion (c'est-à-dire que l'utilisateur pourra accéder à l'application web sans aucune connectivité).

Une Progressive Web App s'appuie donc sur plusieurs principes :

* Sécurisée : L'application doit être sécurisée, c'est-à-dire accessible via HTTPS. Certaines API HTML ne sont d'ailleurs accessibles que via HTTPS (Service Worker, Geolocation, etc.)
* Responsive : L'application est accessible depuis plusieurs devices, elle doit donc être responsive afin de s'adapter à toutes les tailles d'écran
* Rapide et accessible offline :  L'application doit être rapide à charger et accessible sans connexion. Pour cela, on peut s'appuyer sur les Service Worker comme vous pouvez le voir ici <a href="https://blogs.infinitesquare.com/posts/web/service-worker-rendre-votre-site-accessible-meme-sans-connexion" target="_blank">Service Worker : Rendre votre site accessible même sans connexion !</a>.
* Notification push :  L'application peut recevoir des notifications push, même si le navigateur est fermé et que l'application web n'est pas active. Cette fonctionnalité est possible grâce à la Push API, comme nous le verrons également dans un prochain article.

![Demo Push API](https://sebastienollivier.blob.core.windows.net/blog/progressive-web-app-ou-comment-se-passer-des-stores/push-demo.gif =345x300 "Demo Push API")

* Accessible depuis l'écran d'accueil : L'utilisateur doit pouvoir créer un raccourci vers l'application web, grâce au Web Manifest, comme nous le verrons enfin dans un dernier article.

![Demo WebManifest](https://sebastienollivier.blob.core.windows.net/blog/progressive-web-app-ou-comment-se-passer-des-stores/webmanifest-demo.gif" =248x300 "WebManifest")

Google propose une liste regroupant un ensemble d'éléments permettant de créer une Progressive Web App : <a title="Progressive Web App Google checklist" href="https://developers.google.com/web/progressive-web-apps/checklist" target="_blank">https://developers.google.com/web/progressive-web-apps/checklist</a>. Si vous utilisez Chrome, les outils de développements (F12) intègrent une fonctionnalité d'audit via l'onglet Audits qui permet d'inspecter la page courante afin de ressortir une analyse complète (cet outil est également disponible sous la forme d'une extension Chrome : <a href="https://developers.google.com/web/tools/lighthouse/" target="_blank">Lighthouse</a>).

![Audit de Chrome](https://sebastienollivier.blob.core.windows.net/blog/progressive-web-app-ou-comment-se-passer-des-stores/audit-chrome.png =879x178 "Audit de Chrome")

A vos claviers, l'avenir est à nous !

