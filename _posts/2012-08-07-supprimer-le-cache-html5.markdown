---
layout: post
title: 'Supprimer le cache HTML 5'
date: 2012-08-07
categories: html5
resume: 'Quelles sont les solutions pour demander la supprimer le cache HTML 5 du navigateur depuis votre serveur ?'
tags: cache html5 clear
---
L'une des nouveautés les plus importantes introduites avec l'arrivée de HTML 5 est la possibilité de créer des applications web possédant de vraies fonctionnalités offline.

En combinant le cache HTML 5 (permettant la navigation offline) au Local Storage HTML 5 (permettant le stockage local sous forme de clef / valeur, uniquement accessible par le navigateur), on peut proposer à l'utilisateur de travailler hors connexion, en stockant ses actions dans le Local Storage puis en synchronisant les données stockées une fois la connexion rétablie.

Le sujet de ce post n'étant pas de montrer comment créer une application web offline, voici un lien expliquant le fonctionnement du cache HTML 5 : <a href="http://www.html5rocks.com/en/tutorials/appcache/beginner/" target="_blank">http://www.html5rocks.com/en/tutorials/appcache/beginner/</a> et voici un autre lien expliquant comment l'implémenter dans un contexte ASP.NET MVC : <a href="http://deanhume.com/Home/BlogPost/mvc-and-the-html5-application-cache/59" target="_blank">http://deanhume.com/Home/BlogPost/mvc-and-the-html5-application-cache/59</a>.

Nous allons voir dans ce post comment demander la suppression du cache HTML 5 du navigateur.

## Authentification et offline

Une des problématiques qui apparaît assez rapidement quand on crée une application web offline est l'authentification, et plus précisément la déconnexion de l'utilisateur.

Généralement, on autorise la déconnexion de l'utilisateur uniquement lorsque celui-ci est online pour s'assurer que tout le travail effectué offline est synchronisé et ainsi ne pas perdre de données.

Une fois toutes les données synchronisées et l'utilisateur déconnecté, il reste à vider le cache HTML 5 du navigateur. En effet, si on ne vide pas le cache du navigateur, un utilisateur pourra naviguer sur toutes les pages de l'application web mise en cache par le navigateur sans avoir à s'authentifier (le cache contenant uniquement des pages HTML statiques, donc pas de gestion d'authentification).

## Invalidation et suppression du cache HTML 5

### <s>Invalidation par Javascript</s>

Javascript propose une API HTML 5 permettant d'intéragir avec les nouvelles features. Malheureusement, cette API ne permet pas de manipuler le cache du navigateur mais uniquement de suivre l'état du cache via un ensemble d'évênements (checking, downloading, updateready etc.). Donc impossible de supprimer le cache via Javascript.

### Envoyer un nouveau manifest

La solution consiste à renvoyer un nouveau manifest contenant uniquement les pages accessibles. Dès que le navigateur détectera une nouvelle version du manifest, il mettra à jour le cache. On peut aussi forcer le navigateur à ne pas utiliser le cache sur certaines pages en les rajoutant dans la partie NETWORK du manifest.

### HTTP 404 / 410

Une solution alternative est de renvoyer un code HTTP 404 ou 410 lors de la requête vers le manifest. W3C spécifie qu'à la réception d'un de ces codes, le cache est déclaré comme obsolète et est supprimé par le navigateur : <a href="http://www.w3.org/TR/2011/WD-html5-20110525/offline.html#downloading-or-updating-an-application-cache" target="_blank">http://www.w3.org/TR/2011/WD-html5-20110525/offline.html#downloading-or-updating-an-application-cache</a>.

Chaque navigateur implémente sa propre logique de cache en respectant les spécifications HTML 5 W3C. Il est donc important de tester en cours de développement le bon fonctionnement des applications offline sur chaque navigateur.

Bon développement offline ;)


