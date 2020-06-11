---
layout: post
title: 'Exemple d’une directive AngularJS utilisant la phase compile'
date: 2014-12-19
categories: angularjs
resume: 'Illustration du fonctionnement de la phase compile des directives AngularJS via la création d''une directive déferrant le chargement d''éléments afin d''optimiser le chargement d''une page.'
tags: angularjs directive compile
---
On trouve pas mal d'exemples sur Internet illustrant le fonctionnement des directives AngularJS, mais on en trouve assez peu illustrant leur phase `compile`.

On va voir dans cet article un exemple de directive utilisant cette phase pour optimiser le chargement d'une page en retardant l'interprétation de certains éléments.

## Application de base

L'application qui va servir à illustrer notre exemple contient une page qui affiche une liste de cent recettes dans une grille. A chaque survol de la souris sur une recette, l'image de cette recette est affichée.

<iframe style="width: 100%; height: 400px" src="https://embed.plnkr.co/tyLCMb/preview"></iframe>

L'application est composée d'un contrôleur initialisant la propriété `recipies` du scope avec un tableau de cent recettes. Chaque recette est définie par une propriété `img`, contenant une url vers l'image de la recette, et une propriété `name`, contenant le nom de la recette.

```js
module.controller("MainController", function($scope) { 
    $scope.recipies = [{name: "Ma Recette", img: "http://localhost/marecette.png"}, …]; 
});
```

La vue utilise la directive `ngRepeat` pour itérer sur la liste des recettes. Pour chaque recette, une `div` contenant le nom de la recette et une `div` contenant son image vont être créées.

```html
<body ng-controller="MainController"> 
    <div ng-repeat="recipy in recipies" class="recipy"> 
        <a href="#"> 
            <div class="main">{{recipy.name}}</div> 
            <div class="detail"> 
                <img ng-src="{{recipy.img}}" /> 
            </div> 
        </a> 
    </div> 
</body> 
```

Les styles css suivants sont appliqués :

```css
a, a:hover { 
    width: 100%; 
    height: 100%; 
    color: black; 
    text-decoration: none; 
} 

a:hover .main { 
    display: none; 
} 

a:hover .detail { 
    display: block; 
} 

.detail { 
    display: none; 
} 

```
La div contenant l'image de la recette est cachée par défaut et est affichée au survol de la zone de la recette.

## Comment optimiser l'affichage de la page ?

L'application précédente fonctionne correctement. La liste des recettes est visible et le survol d'une recette permet d'afficher son image.

Malgré cela, le mécanisme mis en place n'est pas optimisé. Au chargement de la page, pour chaque recette de la liste, la `div` contenant l'image de la recette est interprétée par AngularJS. Ces `div` ne seront affichées qu'au survol de la zone, ce qui veut dire que cent `div` seront interprétées par AngularJS au chargement de la page sans être affichées à l'utilisateur.

Pour optimiser ça, l'idée est de retarder l'interprétation de la zone non affichée au moment où elle le sera. De cette manière, les cents div contenant les images ne seront pas interprétées au chargement de la page mais au survol de la zone de la recette.

## Création de la directive

L'objectif de la directive va être de supprimer la `div` contenant l'image du template, puis de l'injecter au survol d'une recette. Elle sera nommée `deferShowDetail` et sera appliquée sur la balise HTML liée à la directive `ngRepeat` :
```html
<div ng-repeat="recipy in recipies" class="recipy" defer-show-detail> 
    <a href="#"> 
        <div class="main">{{recipy.name}}</div> 
        <div class="detail"> 
            <img ng-src="{{recipy.img}}" /> 
        </div> 
    </a> 
</div>
```

Lors de la phase `compile`, la directive va supprimer la `div` contenant l'image pour qu'elle ne soit pas interprétée par AngularJS :

```js
module.directive("deferShowDetail", function() { 
    return { 
        restrict: 'A', 
        compile: function(element) { 
            element.find("div.detail").remove(); 
        } 
    } 
}); 
```
Dans le but de réinjecter la `div` au survol de la zone, il faut la sauvegarder. La `div` va être sauvegardée compilée, à l'aide du service `$compile`, pour pouvoir être réinjectée plus efficacement :

```js
module.directive("deferShowDetail", function($compile) { 
    return { 
        restrict: 'A', 
        compile: function(element) { 
            var detailTemplate = $compile(element.find("div.detail").remove());
        } 
    } 
}); 
```

Pour enregistrer la position de la `div` dans le template et pouvoir la réinsérer au bon endroit, on va rajouter un commentaire HTML :

```js
module.directive("deferShowDetail", function($compile) { 
    var replacementMarkup = "<!-- defer-show-detail --\>";

    return { 
        restrict: 'A', 
        compile: function(element) { 
            var detailTemplate = $compile(element.find("div.detail").replaceWith(replacementMarkup));
        } 
    } 
}); 
```
La phase compile de la directive doit être exécutée en premier, avant celle de la directive `ngRepeat`. Si ce n'est pas le cas, la directive `ngRepeat` va compiler le template puis le supprimer du DOM. Ensuite la phase compile de la directive `deferShowDetail` va s'exécuter sur le template vide et ne pourra pas effectuer ses actions. On va positionner la priorité de la directive à 2000, pour être supérieure à celle de `ngRepeat` :

```js
module.directive("deferShowDetail", function($compile) { 
    var replacementMarkup = "<!-- defer-show-detail --\>";

    return { 
        restrict: 'A', 
        priority: 2000, 
        compile: function(element) { 
            var detailTemplate = $compile(element.find("div.detail").replaceWith(replacementMarkup));
        } 
    } 
}); 
```

A ce point, la vue générée ne contient plus la div qui a été remplacée par le commentaire :

![Etat du DOM après suppression](https://sebastienollivier.blob.core.windows.net/blog/exemple-directive-compile/dom-before-mousehover.png =560x242 "Etat du DOM après suppression")

La directive va ensuite s'abonner à l'évènement `mouseenter`, déclenché lors du survol, sur toutes les div correspondant à une recette, via la classe CSS `recipy`.

```js
$("body").on("mouseenter", ".recipy", function(e) {
} 
```

Au `mouseenter`, la directive va récupérer le commentaire précédemment inséré pour déterminer la position où injecter la div :
```js
$("body").on("mouseenter", ".recipy", function(e) {
    var target = $(this).find("*").contents().filter(function(){ 
        return this.nodeType == 8 && this.textContent.trim() === "defer-show-detail"; 
    });
} 
```

Si le commentaire n'est plus dans la vue, c'est que la div a déjà été injectée, donc on ne fait rien.

```js
$("body").on("mouseenter", ".recipy", function(e) {
    var target = $(this).find("*").contents().filter(function(){ 
        return this.nodeType == 8 && this.textContent.trim() === "defer-show-detail"; 
    });
    
    if(target.length) { 
        // TODO: injecter la div
    } 
} 
```

Si le commentaire existe, la directive va appliquer le scope courant au template compilé pour retrouver la div à injecter puis va la placer à sa position initiale.

Le scope courant est récupérable en utilisant la méthode `scope` sur l'élément HTML représentant le commentaire :

```js
if(target.length) { 
    var scope = target.scope();
}
```

Ensuite, le template compilé va être utilisé avec le scope pour générer la vue qui remplacera le commentaire HTML :

```js
if(target.length) { 
    var scope = target.scope(); 
    
    detailTemplate(scope, function(detail) { 
       target.replaceWith(detail); 
    }); 
} 
```
La directive est terminée. Au chargement de la page, les div contenant les images des recettes ne sont pas chargées :

![DOM avant mousehover](https://sebastienollivier.blob.core.windows.net/blog/exemple-directive-compile/dom-before-mousehover.png =560x242 "DOM avant mousehover") 

Au survol d'une zone, ce qui correspond dans l'exemple suivant à la zone de la recette du Tiramisu, la directive va injecter le template pour qu'il soit affiché :

![DOM après mousehover](https://sebastienollivier.blob.core.windows.net/blog/exemple-directive-compile/dom-after-mousehover.png =900x376 "DOM après mousehover") 

Lorsque la zone sera de nouveau survolée, la directive ayant déjà injecté la `div` contenant l'image, aucun traitement ne sera effectué.

En retardant l'interprétation de la `div` contenant l'image de la recette, cette directive a permis à AngularJS de n'avoir à interpréter que les éléments à afficher au chargement de la page. Puis en injectant le template précédemment supprimé au survol d'une recette, cette directive rétablit le fonctionnement attendu par l'utilisateur.

Vous trouverez le code de cette directive sur le plnkr suivant : <a href="https://embed.plnkr.co/EhAgXT/preview" target="_blank">Recipies list (with compile directive optimization)</a>.

Bonnes directives !

