---
layout: post
title: 'UrlHelper en AngularJS'
date: 2014-05-20
categories: angularjs
resume: 'Comment créer une abstraction entre l''URL d''une route Angular et son identification (pour ne pas avoir à utiliser l''URL comme identifiant) ?'
tags: angularjs url-helper routes
---
AngularJS utilise la notion de route pour associer à une URL un contrôleur et une vue. L'idée est de créer une abstraction entre l'URL et la vue HTML ciblée.

Voici un exemple de déclaration de routes AngularJS :

```js
module.config(function ($routeProvider) {
    $routeProvider
        .when("/",
        {
            controller: "mainController",
            templateUrl: "app/views/main.html",
        })
        .when("/page1",
        {
            controller: "page1Controller",
            templateUrl: "app/views/page1.html",
        })
        .when("/page1/:param",
        {
            controller: "page1Controller",
            templateUrl: "app/views/page1.html",
        })
        .when("/page2",
        {
            controller: "page2Controller",
            templateUrl: "app/views/page2.html",
        })
        .otherwise({ redirectTo: "/" });
});
```

Au niveau des vues, pour créer des liens vers des routes, il faut renseigner l'URL de la route :

```html
<a href="/page1">Lien vers la page 1</a>
```

Il n'existe pas d'abstraction entre l'URL de la route et la route elle-même. C'est à dire que si on crée des liens vers une route et que l'on change ensuite l'URL de la route, il va falloir parcourir toutes les vues pour s'assurer que les URL ciblant cette route ont bien été mises à jour. Cela implique un travail fastidieux et surtout un risque d'erreurs assez important.

L'objectif de ce post est d'expliquer comment mettre en place un mécanisme permettant de créer cette abstraction (à la manière de Url.Action en ASP.NET MVC).

## Nommer les routes

Avant de commencer, on va nommer les routes. L'idée est de pouvoir identifier une route par un nom et non plus par une url. Pour cela, on va juste rajouter une propriété `name` aux routes :

```js
module.config(function ($routeProvider) {
    $routeProvider
        .when("/",
        {
            name: "main",
            controller: "mainController",
            templateUrl: "app/views/main.html",
        })
        .when("/page1",
        {
            name: "page1",
            controller: "page1Controller",
            templateUrl: "app/views/page1.html",
        })
        .when("/page1/:param",
        {
            name: "page1",
            controller: "page1Controller",
            templateUrl: "app/views/page1.html",
        })
        .when("/page2",
        {
            name: "page2",
            controller: "page2Controller",
            templateUrl: "app/views/page2.html",
        })
        .otherwise({ redirectTo: "/" });
});
```

## Création d'un service de gestion des URL

On va maintenant créer un service dont le rôle est de nous retourner une URL à partir d'un nom de route.

```js
module.factory("urlService", ["$route", "$exceptionHandler", function ($route, $exceptionHandler) {
    var self = this;

    self.routes = _.sortBy(
                    _.map(
                        _.filter($route.routes, function (route) {
                            return route.name;
                        }),
                    function (route) {
                        return { name: route.name, path: route.originalPath, keys: route.keys }
                    }),
                function (route) {
                    return route.keys.length;
                });

    return {
        urlFor: function (routeName, params) {
            var eligibleRoutes = _.filter(self.routes, function (route) {
                return route.name == routeName;
            });

            for (var i = eligibleRoutes.length - 1; i >= 0; i--) {
                var eligibleRoute = eligibleRoutes[i];
                var computedRoute = eligibleRoute.path;
                var match = true;

                for (var j = 0; j < eligibleRoute.keys.length; j++) {
                    var key = eligibleRoute.keys[j].name;

                    if (params && params[key]) {
                        computedRoute = computedRoute.replace(":" + key, params[key]);
                    } else {
                        match = false;
                        break;
                    }
                }

                if (match) {
                    return computedRoute;
                }
            }

            $exceptionHandler("Route '" + routeName + "' not found for parameters : " + params);
        }
    };
}]);
```

Le service expose une méthode `urlFor` qui retourne une url en fonction d'un nom de route et d'une liste de paramètres. Cette méthode va simplement parcourir les routes de l'application (qui ont été triées et ordonnées à la création du service, avec l'aide de la librairie <a href="http://lodash.com/" target="_blank">Lo-Dash</a>), récupérer celle qui correspond au nom et aux paramètres passés puis la renvoyer complétée.


## Filtres et directives

Pour pouvoir utiliser le service dans les vues, on va crééer un filtre et deux directives :


```js
module.filter("url", ["urlService", function (urlService) {
    return function (routeName, params) {
        return urlService.urlFor(routeName, params);
    }
}])
.directive('urlFor', ['urlService', function (urlService) {
    return {
        restrict: 'EA',
        scope: {
          urlForParams : "="
        },
        link: function (scope, element, attrs) {
            var url = urlService.urlFor(attrs.urlFor, scope.urlForParams);

            element.html(url);
        }
    };
}])
.directive('urlForHref', ['urlService', function (urlService) {
    return {
        restrict: 'A',
        scope: {
          urlForParams : "="
        },
        link: function (scope, element, attrs) {
            var url = urlService.urlFor(attrs.urlForHref, scope.urlForParams);
            attrs.$set('href', url);
        }
    };
}]);
```

Ces trois composants permettent de générer une url dans la vue en utilisant le service `urlService`. Le filtre est réévalué à chaque digest ce qui permet de créer des liens dynamiques (utilisant des propriétés du modèle comme paramètres par exemple). Si l'url n'a pas besoin d'être dynamique, il est préférable d'utiliser les directives pour des questions de performances. Dans ce cas, le lien n'est généré qu'une seule fois, lors de la phase link de la directive.

L'utilisation du filtre se fait de la manière suivante :

```html
<p>{% raw %}{{ 'page1' | url:{param: property} }}{% endraw %}</p>

<a href="{% raw %}{{ 'page1' | url:{param: property} }}{% endraw %}">{% raw %}{{property}}{% endraw %}</a>
```

La première directive insère l'url dans l'élement HTML courant, la deuxième directive insère l'url dans l'attribut href de l'élément HTML courant.

```html
<div data-ng-url-for="page1" data-url-for-params="{param: 'value'}" />

<a data-ng-url-for-href="page1" data-url-for-params="{param: 'value'}">My Link</a>
```

## Module Url Helper

Le service, le filtre et les directives ont été packagés dans un module AngularJS. Vous pouvez trouver un exemple d'utilisation sur le Plunker suivant :

<iframe src="https://embed.plnkr.co/USOiq1/preview" style="width: 100%; height: 300px" frameborder="0"></iframe>

Les sources du module se trouvent ici : <a href="https://onedrive.live.com/redir?resid=2BAE7BB2DE0FBFC7%21336" target="_blank">urls.js</a>.

N'hésitez pas à me communiquer vos remarques !


