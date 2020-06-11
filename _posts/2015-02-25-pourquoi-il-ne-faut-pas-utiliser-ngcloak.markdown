---
layout: post
title: 'Pourquoi il ne faut pas utiliser ngCloak ?'
date: 2015-02-25
categories: angularjs
resume: 'ngCloak permet de masquer un élément jusqu''à ce que l''application soit correctement chargée. Mais cette directive peut ne pas suffir. Voyons pourquoi et comment y remédier.'
tags: angularjs ngcloak
---
<a href="https://docs.angularjs.org/api/ng/directive/ngCloak" target="_blank">`ngCloak`</a> est une directive AngularJS qui masque un élément jusqu'à ce que l'application soit correctement chargée. Elle permet par exemple d'éviter que la syntaxe `{% raw %}{{ }}{% endraw %}` des bindings AngularJS ne s'afficher brièvement au chargement de la page.

Dans certains cas, notamment sur des devices mobiles d'entrée de gamme (Android surtout), `ngCloak` ne suffit pas et un effet de clignotement peut apparaître. On va voir dans ce post pourquoi et comment y remédier.
 
## Comment fonctionne ngCloak ?

Lors de son exécution, AngularJS va injecter un ensemble de classes CSS dans la page. Voici les dernières lignes du fichier `angular.js` : 

![AngularJS code source](https://sebastienollivier.blob.core.windows.net/blog/pourquoi-il-ne-faut-pas-utiliser-ngcloak/ng-css.png =606x73 "AngularJS code source")

L'idée est d'appliquer un style `display: none;` à tous les éléments liés à la directive `ngCloak` pour qu'ils soient cachés.

Lors de sa phase compile, cette directive va se supprimer du DOM, ce qui aura comme effet d'afficher l'élément rattaché, puisque la classe CSS ne sera plus appliquée. Voici le code de la directive `ngCloak` :


![ngCloak code source](https://sebastienollivier.blob.core.windows.net/blog/pourquoi-il-ne-faut-pas-utiliser-ngcloak/ng-cloak-directive.png =283x105 "ngCloak code source")

## Pourquoi ça pose problème ?

Le premier problème vient du fait qu'AngularJS injecte les classes CSS lors de son exécution. Entre le chargement de la page et la fin de l'exécution d'AngularJS, le mécanisme de `ngCloak` n'est pas fonctionnel et certaines zones à cacher seront donc visibles. Pour régler cela, c'est plutôt simple, il faut ajouter les styles suivants dans votre CSS :

```css
[ng-cloak], [data-ng-cloak], [x-ng-cloak], .ng-cloak {
	display: none;
}
```

De cette manière, les éléments liés à la directive `ngCloak` seront cachés avant le chargement d'AngularJS.

Le second problème vient du fait que la suppression des attributs `ngCloak` est faite lors de la phase compile de la directive. A ce moment, le contrôleur de la vue n'aurait pas été exécuté et donc le scope ne sera pas initialisé.

Imaginons le contrôleur suivant :

```js
function ListController($scope, businessService) {   
    $scope.isLoading = true;

    businessService.getItems().then(function (items) {
        $scope.items= items;
    }, function () {
        $scope.errorRetrievingItems = true;
    })
    .finally(function () {
        $scope.isLoading = false;
    });
}
```

Et la vue suivante :

```html
<div data-ng-show="isLoading">
        Chargement des données en cours…
</div>
<div data-ng-cloak class="error-content" data-ng-show="items && items.length === 0 && !isLoading">
        <p class="error-text">Aucun élément</p>
 </div>
 <div data-ng-cloak class="error-content" data-ng-show="errorRetrievingItems">
        <p class="error-text">Impossible de récupérer les éléments</p>
 </div>

<ul>
	<li data-ng-repeat="item in items">[…]</li>
</ul>
```

Sur l'émulateur Android de Visual Studio, le résultat est le suivant :

![Démo du ngCloak](https://sebastienollivier.blob.core.windows.net/blog/pourquoi-il-ne-faut-pas-utiliser-ngcloak/demo.gif =261x507 "Démo du ngCloak")

Lorsque la phase compile de la directive `ngCloak` est déclenchée, le contrôleur n'a pas été instancié et le scope n'est donc pas initialisé. Les expressions associées aux directives `ngShow` n'ont pas encore été interprétées et les messages "Aucun élément" et "Impossible de récupérer les éléments" sont visibles.

Une fois que le contrôleur a initialisé le scope et que les expressions ont été interprétées, l'affichage revient à ce qui était attendu.

## Mais comment on fait ?

Il suffit de déférer l'affichage des éléments liés à la directive `ngCloak` après que le contrôleur ait initialisé le scope, donc lors de la phase `postLink` (pour plus d'informations sur les différentes phases d'une directive, vous pouvez vous référer à cet article <a href="http://sebastienollivier.fr/blog/angularjs/cycle-dexecution-des-directives-angularjs">Cycle d'exécution des directives AngularJS</a>). Voici le code de la directive `deferredCloak` (identique à la directive `ngCloak` mais utilisant la phase `postLink`) :

```js
module.directive("deferredCloak", function () {
    return {
        restrict: 'A',
        link: function (scope, element, attrs) {        
            attrs.$set("deferredCloak", undefined);
            element.removeClass("deferred-cloak");
        }
    };
});
```

Les styles CSS suivants sont nécessaires pour cacher les éléments liés à cette directive :

```css
[deferred-cloak], [data-deferred-cloak], [x-deferred-cloak], .deferred-cloak {
	display: none;
}
```

La vue précédente devient donc :

```html
<div data-deferred-cloak data-ng-show="isLoading">
    Chargement des données en cours…
</div>
<div data-deferred-cloak class="error-content" data-ng-show="items && items.length === 0 && !isLoading">
    <p class="error-text">Aucun élément</p>
</div>
<div data-deferred-cloak class="error-content" data-ng-show="errorRetrievingItems">
    <p class="error-text">Impossible de récupérer les éléments</p>
</div>

<ul>
    <li data-ng-repeat="item in items">[…]</li>
</ul>
```

Et voilà, les éléments sont correctement cachés et ne s'affichent que lorsque le scope est correctement initialisé. Plutôt simple.


A bientôt !


