---
layout: post
title: 'Attendre que le DOM soit prêt avant d''exécuter une directive AngularJS'
date: 2016-02-17
categories: angularjs
resume: 'Voici une astuce permettant d''attendre que le DOM soit prêt (càd que le repaint ait été fait) avant d''exécuter la méthode ''postLink'' d''une directive AngularJS.'
tags: angularjs directive dom-ready
---
Lorsque la méthode `postLink` d'une directive AngularJS se déclenche, les directives situées dans l'arborescence descendante du DOM ont déjà été exécutées, c’est-à-dire que les bindings et les transclusions ont déjà eu lieu.

_Pour avoir plus d'informations sur les phases d'exécution des directives AngularJS, vous pouvez vous référer à l'article suivant : <a href="http://sebastienollivier.fr/blog/angularjs/cycle-dexecution-des-directives-angularjs">http://sebastienollivier.fr/blog/angularjs/cycle-dexecution-des-directives-angularjs</a>._

Il est cependant possible que lors de l'exécution du `postLink`, le DOM ne soit pas tout à fait prêt, le browser n'ayant pas eu le temps de faire un repaint. Ce cas de figure peut être assez embêtant lorsque l'on souhaite accéder à des propriétés calculées du DOM, comme la hauteur d'un élément par exemple.

![Execution du link de la directive](https://sebastienollivier.blob.core.windows.net/blog/attendre-dom-ready-directive/sanstimeout.jpg =1108x527 "Execution du link de la directive")

_Dans l'exemple précédent, la hauteur de l'élément n'est pas la bonne, puisque le browser n'a pas eu le temps de mettre à jour le DOM._

L'astuce pour attendre que le DOM soit totalement prêt est assez simple. Il suffit d'encapsuler le contenu du postLink dans un `setTimeout` de 0 et le tour est joué :

![Execution du link de la directive avec timeout](https://sebastienollivier.blob.core.windows.net/blog/attendre-dom-ready-directive/avectimeout.jpg =1159x527 "Execution du link de la directive avec timeout")

Si vous avez besoin de déclencher l'exécution d'un cycle digest (pour mettre à jour les bindings), vous pouvez utiliser le service `$timeout` d'AngularJS.

Les meilleures recettes sont souvent les plus simples !

