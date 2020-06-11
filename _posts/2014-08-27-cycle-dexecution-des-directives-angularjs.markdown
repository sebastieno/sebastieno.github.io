---
layout: post
title: 'Cycle d''exécution des directives AngularJS'
date: 2014-08-27
categories: angularjs
resume: 'Les directives sont le coeur d''une application AngularJS mais sont assez difficiles à comprendre. Venons découvrir ici comment elles sont exécutées et quel est leur cycle de vie.'
tags: angularjs directive lifecycle compile prelink postlink link
---
Bien qu'assez difficiles à appréhender, les directives d'AngularJS offrent d'énormes possibilités. Elles permettent de créer des composants réutilisables, de manipuler le DOM en respectant la séparation des responsabilités définies par l'architecture d'AngularJS ou d'enrichir la syntaxe HTML comme le fait `ngIf`, `ngShow` ou bien `ngRepeat`.

Dans cet article, nous allons voir comment AngularJS exécute les directives.

## Cycle d'exécution des directives

L'exécution des directives se fait en plusieurs phases, lors desquelles la directive peut agir.

### Parcours du DOM

A chaque fois qu'un template de vue doit être affiché, AngularJS va dans un premier temps parcourir le DOM de ce template pour découvrir les directives utilisées. L'association entre la directive et son élément HTML rattaché va alors être enregistrée.

### Compile

La seconde phase est la phase compile. Lors de celle-ci, le template HTML associé à la directive a été modifié selon la configuration de la directive, notamment selon les propriétés `template` / `templateUrl` et `replace`.
Le scope n'est pas créé et la transclusion n'est pas encore effectuée.

Cette phase est assez rarement utilisée par les directives. Elle permet d'agir sur le template avant qu'il ne soit interprété.

### Controller

Le contrôleur de la directive est ensuite créé. Un contrôleur de directive, à ne pas confondre avec un contrôleur classique, permet d'exposer des méthodes et des propriétés à d'autres directives, de manière à pouvoir établir une communication entre directives. La directive `ng-model` par exemple déclare un contrôleur exposant notamment la méthode `$setViewValue` aux autres directives.

Un contrôleur est créé pour chaque instance d'une même directive. C'est-à-dire que si le template associé à une directive est dupliqué, ce qui est le cas pour les éléments situés dans la directive `ng-repeat`, chaque duplication de la directive aura son propre contrôleur.

### Link

Ensuite vient la phase link. Cette phase correspond au moment où le template est lié au scope et se divise en deux sous-phases.

#### PreLink

Lors de la phase preLink, le scope a été associé au template mais la transclusion n'a pas encore été effectuée.  Cette phase est déclenchée une fois par instance de la directive et est exécutée avant que les directives situées dans l'élément HTML associé à la directive ne soient exécutées.

Cette phase est généralement utilisée pour enrichir le scope courant, avant que les directives enfants ne soient exécutées.

#### PostLink

La dernière phase est la phase postLink. Lors de cette phase, la transclusion a été effectuée et l'interprétation des expressions, notamment les bindings, peut se faire. Elle est aussi exécutée une fois par instance de la directive.

C'est lors de cette phase que la majorité des directives vont effectuer un traitement pour s'abonner aux évènements du DOM, pour déclarer des `$watch`, etc.

## Ordre d'exécution des phases

Chacune des phases s'exécute de manière ordonnée.

La phase compile s'exécute une fois par déclaration d'une directive, dans l'ordre de leur apparition dans le DOM. Si deux directives sont situées sur le même élément HTML, la directive possédant la propriété `priority` la plus élevée vera sa phase compile exécutée la première.

Ensuite, les phases controller et pre-link s'exécutent successivement sur chacune des directives, dans l'ordre de leur apparition dans le DOM. Si deux directives sont situées sur le même élément HTML, la directive possédant la propriété `priority` la plus élevée vera sa phase controller puis sa phase pre-link exécutées en premier.

Enfin, la phase post-link d'une directive s'exécute une fois que toutes les phases de toutes les directives situées dans l'arborescence descendante du DOM ont été exécutées. Si deux directives sont situées sur le même élément HTML, la directive possédant la propriété `priority` la plus faible vera sa phase post-link exécutée en premier.

## Illustration sur un exemple

Pour illustrer le cycle d'exécution des directives, on va utiliser le template suivant :

```html
<ul ng-controller="MainController">
    <li ng-repeat="name in names" low-directive high-directive>
        <p hello-message>{{name}}</p>
    </li>
</ul>
```

Le template contient la directive `ng-repeat` associée à l'élément `li`. Sur ce même élément, on retrouve aussi les directives `low-directive` et `hight-directive`, correspondant à des directives ayant respectivement une priorité à 0 et à 2000 (la directive `ng-repeat` ayant une priorité de 1000). Dans le contenu du `li`, la directive `hello-message` est associée à un élément `p`. Cette directive utilise le mécanisme de transclusion pour afficher un message de bienvenue via le template suivant :

```js
template: "Bonjour <span ng-transclude></span> !"
```

### Parcours du DOM

Au chargement de la page, l'association entre les directives `ng-repeat`, `low-directive` et `high-directive` et l'élément `li` va être enregistrée. L'association entre l'élément `p` et la directive `hello-message` va aussi être enregistrée. Le DOM n'a pour l'instant pas encore été modifié :

![Template initial](https://sebastienollivier.blob.core.windows.net/blog/ng-directives-execution-flow/initial-template.png =444x69 "Template initial")

### Compile

Ensuite, la phase compile est exécutée sur les quatre directives précédentes. Le template de la directive `hello-message` est injecté dans l'élément `p` :

![Template de la directive helloMessage après la tranclusion](https://sebastienollivier.blob.core.windows.net/blog/ng-directives-execution-flow/hellomessage-template-withtranclusion.png =217x69 "Template de la directive helloMessage après la tranclusion")

La directive `ng-repeat` a enregistré le template associé à l'élément `li` puis l'a supprimé du DOM. Ce mécanisme est spécifique à cette directive et lui permet de sauvegarder un template pour pouvoir l'appliquer autant de fois qu'il y a d'éléments sur la collection associée, et de pouvoir le réappliquer lors de l'ajout d'éléments à cette collection. A la fin de la phase compile, le DOM ressemble alors à : 

![Template après la phase compile de ngRepeat](https://sebastienollivier.blob.core.windows.net/blog/ng-directives-execution-flow/template-after-ngrepeat-compile.png =383x47 "Template après la phase compile de ngRepeat")

### Exécution des directives high-directive et ng-repeat

Les phases controller, prelink et postlink vont alors s'exécuter successivement pour les directives `high-directive` et `ng-repeat`. La directive `low-directive` est ignorée car `ng-repeat` utilise le mécanisme de terminal. Ce mécanisme permet, lorsqu'une directive a sa propriété `terminal` à `true`, de ne pas exécuter les phases controller, prelink et postlink des directives associées au même élément HTML et ayant une priorité plus faible.

Après sa phase postlink, la directive `ng-repeat` va injecter le template précédemment enregistré autant de fois qu'il y a d'éléments sur la collection itérée (pour l'exemple, la collection contient deux éléments).

### Première injection du template

Une première injection du template est effectuée. Le premier élément de la collection est défini en tant que scope du template injecté. Le DOM est alors le suivant :

![DOM après première injection du template"](https://sebastienollivier.blob.core.windows.net/blog/ng-directives-execution-flow/template-with-firstrow.png =560x148 "DOM après première injection du template")

Les phases controller puis prelink de la directive `low-directive` vont pouvoir s'exécuter pour cette itération du template. Les directives `high-directive` et `ng-repeat` sont ignorées car elles ont été traitées précédemment. Ensuite les phases controller, prelink puis postlink vont se déclencher sur la directive `hello-message`. A ce moment, la tranclusion a été effectuée et le DOM ressemble à : 

![DOM après première injection du template et transclusion](https://sebastienollivier.blob.core.windows.net/blog/ng-directives-execution-flow/first-row-beforepostlink-hellomessage.png =559x171 "DOM après première injection du template et transclusion")

Enfin, la directive `low-directive` voit sa phase postlink exécutée.

### Seconde injection du template

Une deuxième injection du template est effectuée, avec pour scope le deuxième élément de la collection. Le DOM ressemble alors à :

![DOM après seconde injection du template](https://sebastienollivier.blob.core.windows.net/blog/ng-directives-execution-flow/template-with-secondrow.png =605x170 "DOM après seconde injection du template")

L'exécution des différentes phases des directives `low-directive` et `hello-message` est déclenchée sur la seconde injection, de la même manière que pour la première.

Le traitement de la page est alors terminé.

### Trace de l'exécution des différentes phases

Voici une copie d'écran des traces laissées par l'exécution des différentes phases (le script AngularJS a été légèrement modifié pour que la directive `ng-repeat` trace chacune de ses phases). En rouge, on retrouve les phases compile et en vert les phases exécutées pour chacun des templates injectés.

![Traces de l'exécution des différentes phases](https://sebastienollivier.blob.core.windows.net/blog/ng-directives-execution-flow/traces.png =179x377 "Traces de l'exécution des différentes phases")

Vous pouvez retrouver le code précédent et jouer avec ici : <a href="http://embed.plnkr.co/gG7Ut0/" target="_blank">Plunkr</a>.

Si vous avez des remarques/questions/corrections, n'hésitez pas à me contacter.

Bonne exécution de directives !


