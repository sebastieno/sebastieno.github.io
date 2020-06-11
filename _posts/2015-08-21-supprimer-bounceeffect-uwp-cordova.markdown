---
layout: post
title: 'Supprimer le bounce effect des apps UWP créées avec Cordova'
date: 2015-08-21
categories: cordova
resume: 'Comment peut-on supprimer l''effet ''bounce'' des applications UWP créées avec Cordova à l''aide de CSS ?'
tags: bounce bounce-effect uwp css cordova
---
Lorsque l'on utilise Cordova pour créer une _Universal App_, l'application générée possède un effet de rebond (_bounce effect_). Vous pouvez apercevoir cet effet dans le gif ci-dessous :

![Bounce effect](https://sebastienollivier.blob.core.windows.net/blog/supprimer-bounceeffect-uwp-cordova/bounceeffect.gif =219x400 "Bounce effect")

Même si cela peut paraître peu gênant, ce type de détail trahit l'application en exposant le fait qu'il ne s'agit pas d'une application native, et donne donc un a priori négatif. On va voir dans cet article comment supprimer cet effet à l'aide de CSS.

Le principe consiste à supprimer le scrolling sur toute la page puis à le réactiver sur les éléments que l'on souhaite rendre scrollable.

Il faut d'abord ajouter le style suivant :

```css
html, body {
    height: 100%;
    overflow: hidden;
}
```

Ensuite, l'élément devant être scrollable devra posséder la classe suivante :

```css
.scrollable-content {
    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;
    overflow-y: auto;
}
```

En étant positionné en _absolute_ (pensez à modifier les valeurs des propriétés _top_, _bottom_, _left_ et _right_ pour ajuster le positionnement de l'élément à votre layout) et en ayant _overflow-y_ à _auto_, l'élément deviendra scrollable si son contenu est plus grand que sa taille.

Voici le résultat :

![Sans bounce effect](https://sebastienollivier.blob.core.windows.net/blog/supprimer-bounceeffect-uwp-cordova/withoutbounceeffect.gif =220x400 "Sans bounce effect")

Pas plus compliqué que ça :).


