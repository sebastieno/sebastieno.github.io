---
layout: post
title: 'Optimiser l''affichage de vos sites en différant le chargement des images'
date: 2015-09-14
categories: javascript
resume: 'De nombreuses optimisation permettent d''améliorer le temps d''affichage d''un site. Voyons aujourd''hui une optimisation consistant à différer le chargement des images.'
tags: javascript image-loading optimization
---
De nombreuses études ont démontré que le taux de satisfaction d'un visiteur est fortement lié au temps de chargement d'un site Web (_plus de 50% des utilisateurs abandonnent le site après plus de 3 secondes d'attentes_, selon des études de Akamai ou de Google). Malgré ce constat et malgré nos connexions de plus en plus puissantes (fibre, 4G, etc.), de nombreux sites sont aujourd'hui encore très lent à charger.

Pourtant, il existe de nombreuses optimisations relativement simple à mettre en place et permettant de réduire ce temps de chargement (_cache côté serveur_, _bundle_ et _minification_ des fichiers js et css, _chargement différé du JavaScript non critique_, etc.).

Je vous propose dans cet article de découvrir une de ces optimisations, basée sur le chargement différé d'images.

## Principe
Le principe de cette optimisation est de charger les images non visibles par l'utilisateur (en dessous de la ligne de flottaison) uniquement lorsque la page sera chargée (techniquement au `onload`). De cette manière, on évite de ralentir la page en chargeant des images que l'utilisateur ne pourra pas voir immédiatement.

![Ligne de flottaison](https://sebastienollivier.blob.core.windows.net/blog/differer-chargement-images/ligneflottaison.jpg =186x400 "Ligne de flottaison")

On peut pousser ce principe plus loin en décidant de différer le chargement de toutes les images non critiques (c’est-à-dire non indispensable au sens du 
contenu), même visible.

## Comment faire ?

Pour qu'une image ne soit pas chargée, il faut que le contenu de son attribut _src_ soit vide. L'idée est donc de mettre l'URL de l'image dans un autre attribut, afin d'éviter qu'elle ne soit chargée tout en gardant une référence vers l'URL. Voici un exemple de markup correspondant :

```html
<img src="" data-src="" />
```

Un script JavaScript va ensuite se charger, au `onload`, de parcourir toutes les images de la page et de modifier la valeur des attributs `src`. De cette manière, le chargement de l'image ne se fera qu'une fois le chargement de la page terminé.

```js
function loadImages() {
    var imgs = document.getElementsByTagName('img');
    for (var i = 0; i < imgs.length; i++) {
        if (imgs[i].getAttribute('data-src')) {
            imgs[i].setAttribute('src', imgs[i].getAttribute('data-src'));
        }
    }
}

if (window.addEventListener) { window.addEventListener("load", loadImages, false); }
else if (window.attachEvent) { window.attachEvent("onload", loadImages); }
```

Si vous souhaitez ajouter un overlay à l'image avant qu'elle ne soit chargée (pour indiquer à l'utilisateur qu'il y a une image), vous pouvez appliquer le style suivant :

```css
img[data-src] {
    min-height: 60px;
    background: #EEE url(data:image/png;base64,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) no-repeat center;
}
```

Une image, encodée en base64 pour éviter une requête supplémentaire, est appliquée en background. Ce css donne le résultat suivant :

<style>
.1 {
    min-height: 60px;
    background: #eee url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEwAAAA3CAQAAAAMe2SiAAAHdklEQ…FG0+tZsKPdJY2WG3vuko7v9u247iuP44b32O7Ej+m/CP4PoNJ4zL2IfesAAAAASUVORK5CYII=) no-repeat 50%;
}
</style>

<img class="1" src data-src="" alt="Placeholder" />

## Et qu'est-ce que ça change ?

Pour illustrer le gain potentiel lié à cette optimisation, j'ai effectué quelques mesures à l'aide de l'outil <a href="http://www.webpagetest.org/" target="_blank">WebPageTest</a> sur deux de mes articles : <a href="http://sebastienollivier.fr/blog/cordova/scanner-un-qrcode-avec-cordova">Scanner un QR Code avec Cordova</a> et <a href="http://sebastienollivier.fr/blog/cordova/supprimer-bounceeffect-uwp-cordova">Supprimer le bounce effect des apps UWP créées avec Cordova</a>.

Pour le premier article, avant l'optimisation, les mesures donnaient cela :

![Mesure 1](https://sebastienollivier.blob.core.windows.net/blog/differer-chargement-images/mesure-1.png =862x104 "Mesure 1")

Après optimisation, on arrive à cela :

![Mesure 1 après optimisation](https://sebastienollivier.blob.core.windows.net/blog/differer-chargement-images/mesure-1-optimise.png =862x104 "Mesure 1 après optimisation")

On constate que sur le premier affichage, la page met 0,5 secondes de moins à se charger. Sur le second affichage, le gain se fait moins ressentir (les ressources ont été mises en cache par le navigateur).

Pour le second article, le résultat est encore plus flagrant (il y a deux gifs plutôt gros). Avant optimisation, on a :

![Mesure 2](https://sebastienollivier.blob.core.windows.net/blog/differer-chargement-images/mesure-2.png =869x105 "Mesure 2")

Après optimisation, on arrive à cela :

![Mesure 2 après optimisation](https://sebastienollivier.blob.core.windows.net/blog/differer-chargement-images/mesure-2-optimise.png =866x105 "Mesure 2 après optimisation")

On gagne ici environ 2,5 secondes.

Evidemment, cette optimisation doit être utilisée conjointement à d'autres pour que cela se fasse réellement ressentir (optimisation de la taille des images, inline du css critique, etc.).

Bonne optimisation !
