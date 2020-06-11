---
layout: post
title: 'WinJS pour le Web !'
date: 2014-04-07
categories: javascript
resume: 'WinJS, initialement développé pour les applications Windows, est maintenant compatible pour le Web. Comment ça marche ? Venez le découvrir.'
tags: winjs
---
Voilà une bonne nouvelle pour tous les développeurs Web : WinJS devient compatible pour le Web ! Ce qui veut dire que désormais, WinJS pourra être interprété par tous les navigateurs et tous les devices.

Pour rappel, WinJS est une librairie Javascript qui a été créée pour Windows 8. Cette librairie, associée aux langages HTML et CSS, permettent aux développeurs Web de pouvoir créer facilement des applications Windows 8, en utilisant un langage proche du langage Web (même s'il y a quelques spécificités compte tenu du contexte).

Ensuite, on a pu deviner la volonté de ne plus uniquement se limiter à Windows 8 avec l'arrivée de la version WinJS pour Xbox One, puis la version 2.0 ciblant Windows 8.1 et la version 2.1 ciblant Windows Phone 8.1.

Et maintenant, WinJS devient disponible pour le Web.

Pour essayer, rien de plus simple, il suffit de vous rendre sur <a href="http://try.buildwinjs.com/" target="_blank">http://try.buildwinjs.com/</a> et de tester chaque contrôle (il est même possible d'éditer le code à la volée).

<h1>Comment ça fonctionne ?</h1>

Pour utiliser WinJS sur une application web, il faut commencer par référencer deux fichiers Javascript et un fichier CSS :

```html
<link rel="stylesheet" type="text/css" href="http://try.buildwinjs.com/lib/winjs/css/ui-light.css" />
		
<script type="text/javascript" src="http://try.buildwinjs.com/lib/winjs/js/base.min.js"></script>
<script type="text/javascript" src="http://try.buildwinjs.com/lib/winjs/js/ui.min.js"></script>		
```

Ensuite, en fonction du contrôle que l'on va souhaiter intégrer, il va falloir se référer à la documentation en ligne : <a href="http://try.buildwinjs.com/" target="_blank">http://try.buildwinjs.com/</a>. Si, par exemple, on veut utiliser le contrôle <a href="http://try.buildwinjs.com/default.aspx#appbar" target="_blank">`AppBar`</a>, il faudra rajouter la syntaxe suivante à notre page :

```html
<div data-win-control="WinJS.UI.AppBar" data-win-options="{sticky: true, placement: 'top'}">
    <button data-win-control="WinJS.UI.AppBarCommand" data-win-options="{id:'cmdAdd',label:'Add',icon:'add',section:'global',tooltip:'Add item'}"></button>
    <button data-win-control="WinJS.UI.AppBarCommand" data-win-options="{id:'cmdRemove',label:'Remove',icon:'remove',section:'global',tooltip:'Remove item'}"></button>
    <hr data-win-control="WinJS.UI.AppBarCommand" data-win-options="{type:'separator',section:'global'}" />
    <button data-win-control="WinJS.UI.AppBarCommand" data-win-options="{id:'cmdDelete',label:'Delete',icon:'delete',section:'global',tooltip:'Delete item'}"></button>
    <button data-win-control="WinJS.UI.AppBarCommand" data-win-options="{id:'cmdCamera',label:'Camera',icon:'camera',section:'selection',tooltip:'Take a picture'}"></button>
</div>
```

Les attributs `data-win-control` permettent de renseigner le type de contrôle représenté par l'élément HTML. Dans notre exemple, `WinJS.UI.AppBar` va nous permettre de créer une AppBar et `WinJS.UI.AppBarCommand` de définir une commande de l'AppBar. L'attribut `data-win-options` permet de spécifier les options du contrôle, comme son id, son label ou son icône.

Ensuite, il faut appeler la méthode `processAll()`, pour indiquer à WinJS d'interpréter le HTML, de la manière suivante :

```js
WinJS.UI.processAll().done(function () {
    var appBar = document.getElementById("createAppBar").winControl;
    appBar.show();
});
```

Le callback de la méthode `processAll()` de l'exemple précédent permet d'afficher l'AppBar une fois l'interprétation du HTML terminée par WinJS. Et voilà le résultat :

![AppBar WinJS](https://sebastienollivier.blob.core.windows.net/blog/winjs-pour-le-web/appbar.png =574x86 "AppBar WinJS")

Plutôt convainquant ! Je vous invite à aller voir et à tester les différents contrôles de WinJS : <a href="http://try.buildwinjs.com/" target="_blank">http://try.buildwinjs.com/</a>.

# What's next ?

WinJS devient aussi open source : <a href="https://github.com/winjs/winjs" target="_blank">https://github.com/winjs/winjs</a>. C'est une très bonne nouvelle puisqu'on va tous pouvoir s'investir et apporter notre pierre à l'édifice, que ce soit en donnant des feedbacks, en corrigeant des bugs ou même en créant de nouvelles features.

Il est prévu que WinJS devienne compatible avec les grands frameworks JavaScript de binding (Knockout et AngularJS) ce qui est une autre bonne nouvelle.

Evidemment, WinJS pour le Web est tout nouveau donc il reste pas mal de correctifs à faire pour que ce soit supporté correctement sur chaque navigateur (et de manière responsive sur chaque device). De la même manière, le documentation n'est pas encore très étoffée, mais l'ensemble semble prometteur.

A nous de jouer !


