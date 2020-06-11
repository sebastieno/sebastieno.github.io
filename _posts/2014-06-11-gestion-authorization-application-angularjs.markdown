---
layout: post
title: 'Gestion de l''Authorization dans une application AngularJS'
date: 2014-06-11
categories: angularjs
resume: 'Découvrons comment sécuriser notre application AngularJS en y implémentant un mécanisme d''authorization.'
tags: angularjs authorization
---
Par défaut, AngularJS n'embarque pas de mécanisme permettant de gérer les autorisations, à savoir sécuriser les pages sur lesquelles l'utilisateur doit être authentifié pour pouvoir y accéder.

Dans ce post, on va voir comment mettre en place ce mécanisme, de manière générique, dans une application AngularJS. La gestion des autorisations va se faire sur trois niveaux :

* Rediriger l'utilisateur sur la page de login dès qu'on reçoit un code 401 (Unauthorized) lors d'une requête Ajax
* Rediriger l'utilisateur sur la page de login s'il essaye d'accéder à une page protégée sans être connecté
* Cacher certains éléments si l'utilisateur n'est pas connecté

## Service de gestion de l'utilisateur

Les mécanismes d'authentification d'un utilisateur dépendent du scénario de l'application (authentification par login / mot de passe, OAuth, etc.). Pour la suite du post, on va créer un service responsable de la gestion de l'authentification de l'utilisateur (charge à vous d'implémenter ce service en fonction de vos contraintes): 

```js
module.service("userService", function($rootScope, $http) {
    return {
        isConnected: function() {
            // ...
        },
        signIn: function() {
            // ...
            $rootScope.$broadcast("connectionStateChanged");
        },
        signOut: function() {
            // ...
            $rootScope.$broadcast("connectionStateChanged");
        }
    };
});
```

Le service expose 3 méthodes. `isConnected` renvoit une valeur booléenne permettant de savoir si l'utilisateur est connecté ou non. Les méthodes `signIn` et `signOut` permettent respectivement à un utilisateur de se connecter ou de se déconnecter. Le point important dans ces deux méthodes est qu'elles broadcastent l'évènement `connectionStateChanged` dès que l'utilisateur se connecte ou se déconnecte.

## Redirection sur la page de login si 401

Le code HTTP 401 indique qu'une authentification est requise pour accéder à une ressource Web. Si l'on reçoit ce code en retour d'une requête HTTP, c'est que l'utilisateur n'est pas ou n'est plus authentifié. Dans ce cas, il faut le rediriger vers la page de connexion.

Pour faire cela en AngularJS, il faut simplement créer un intercepteur sur le service `$http` et vérifier le code HTTP en cas d'erreur :

```js
module.config(function ($httpProvider) {
    $httpProvider.interceptors.push(function ($location) {
        return {
            'responseError': function (rejection) {
                if (rejection.status === 401) {
                    $location.url('/connexion?returnUrl=' + $location.path());
                }
            }
        };
    });
});
```

A la réception d'un code d'erreur, on vérifie s'il s'agit d'un 401 et si c'est le cas, on redirige vers la route `/connexion` (on peut rajouter un paramètre `returnUrl` dans l'url pour rediriger l'utilisateur vers la page initiale après sa connexion).

## Sécurisation des routes

Le deuxième point concerne la sécurisation des routes. L'idée est de pouvoir définir quelles routes nécessitent une authentification et de rediriger l'utilisateur vers la page de connexion s'il essaye d'y accéder sans être connecté.

Pour cela, on va ajouter une propriété aux routes permettant de les sécuriser :

```js
module.config(function ($routeProvider) {
    $routeProvider
        .when("/anonymous",
        {
            controller: "anonymousController",
            templateUrl: "app/views/anonymousPage.html"
        })
        .when("/connected",
        {
            controller: "connectedController",
            templateUrl: "app/views/connectedPage.html",
            authorized: true
        })
        .otherwise({ redirectTo: "/anonymous" });
});
```

La route `connected` possède la propriété `authorized` à true indiquant que l'utilisateur doit être authentifié pour pouvoir y accéder.

Pour sécuriser ces routes, on va s'abonner à l'évènement `$routeChangeStart` déclenché à chaque début de changement de route, puis on va vérifier que l'utilisateur soit connecté si la route est sécurisée :

```js
module.run(function($rootScope, $location, userService) {
    $rootScope.$on("$routeChangeStart", function (event, next, current) {
        if (next.$$route.authorized  && !userService.isConnected()) {
            $location.url("/connexion?returnUrl=" + $location.path());
        }
    });
});
```

Si un utilisateur non connecté essaye d'accéder à une route sécurisée, on fera une redirection vers la page de connexion.

## Afficher / Cacher des éléments

Le dernier point que l'on va évoquer consiste à cacher ou afficher des éléments en fonction de l'état de connexion d'un utilisateur. Pour cela, on va créer 2 directives qui vont réagir à l'évènement `connectionStateChanged`: 

```js
module.directive("showWhenConnected", function (userService) {
    return {
        restrict: 'A',
        link: function (scope, element, attrs) {
            var showIfConnected = function() {
                if(userService.isConnected()) {
                    $(element).show();
                } else {
                    $(element).hide();
                }
            };

            showIfConnected();
            scope.$on("connectionStateChanged", showIfConnected);
        }
    };
});
```

```js
module.directive("hideWhenConnected", function (userService) {
    return {
        restrict: 'A',
        link: function (scope, element, attrs) {
            var hideIfConnected = function() {
                if(userService.isConnected()) {
                    $(element).hide();
                } else {
                    $(element).show();
                }
            };

            hideIfConnected();
            scope.$on("connectionStateChanged", hideIfConnected);
        }
    };
});
```

Les directives vont simplement s'abonner à l'évènement `connectionStateChanged` et afficher ou cacher l'élément HTML courant, à l'aide des méthodes `show` et `hide` de `jQuery`, en fonction de l'état de connexion de l'utilisateur.

L'utilisation de ces directives donnent :

```html
<header>
    <div data-show-when-connected>
        Bonjour {{userName}} !
    </div>
    <div data-hide-when-connected>
        <a href="/connexion">Se connecter</a>
        | <a href="/inscription">S'inscrire</a>
    </div>
</header>
```

Voilà comment on peut simplement et rapidement sécuriser une application AngularJS. Evidemment, il ne faut pas oublier qu'une application AngularJS se trouve côté client, dans le navigateur, ce qui implique que le code de l'application peut être modifié assez facilement. La sécurisation d'applications passe donc avant tout par la sécurisation des API Web (ne pas renvoyer d'informations sensibles si l'utilisateur n'est pas authentifié, etc.).

En espérant que tout ça puisse vous aider !

