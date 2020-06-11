---
layout: post
title: 'Créer des validateurs avec AngularJS'
date: 2015-01-07
categories: angularjs
resume: 'Découvrons comment créer des validateurs AngularJS 1.3 (et supérieur) en passant par la propriété $validators.'
tags: angularjs validators
---
Depuis AngularJS 1.3, il n'est plus nécessaire de passer par les `$formatters` et `$parsers` du contrôleur de la directive `ngModel` pour créer des validateurs. Ce contrôleur expose maintenant une propriété `$validators` qui simplifie grandement l'écriture des règles de validations.

On va voir dans cet article comment cela se passe.

## Création d'un validateur

Pour créer un validateur, il faut déclarer une directive devant être utilisée conjointement à la directive <a href="https://docs.angularjs.org/api/ng/directive/ngModel" target="_blank">`ngModel`</a> :

```js
angular.directive("monValidator", function () {
    return {
        require : 'ngModel',
        restrict: 'A'
    };
});
```

Ensuite, dans sa fonction `link`, il faut déclarer une propriété sur la propriété `$validators` du contrôleur de la directive `ngModel` :

```js
angular.directive("monValidator", function () {
    return {
        require : 'ngModel',
        restrict: 'A',
        link: function (scope, element, attrs, ngModel) {
            ngModel.$validators.monValidateur = function(value) {

            };
        }
    };
});
```

Cette propriété doit correspondre à une fonction prenant en paramètre la valeur à valider et retournant un booléen indiquant si cette valeur est valide. Le nom de la propriété sera utilisé par AngularJS pour définir la clef du validateur (utilisable ensuite dans les vues). Voici un exemple de directive permettant de valider qu'un numéro de téléphone est composé de 10 chiffres et commence par un 0 :

```js
angular.directive("phoneValidator", function () {
    return {
        require : 'ngModel',
        restrict: 'A',
        link: function (scope, element, attrs, ngModel) {
            ngModel.$validators.phone = function(value) {
                return !value || /0[\d]{9}/.test(value);
            };
        }
    };
});
```

Il est également possible de créer des validateurs asynchrones via la propriété _$asyncValidators_. Dans ce cas, la fonction doit renvoyer une promise, la résolution de cette promise indique une valeur valide alors qu'un rejet indique une valeur invalide. Voici un exemple de directive vérifiant qu'un login n'est pas encore utilisé :

```js
angular.directive("uniqueUsernameValidator", function ($q, $timeout) {
    return {
        require : 'ngModel',
        restrict: 'A',
        link: function (scope, element, attrs, ngModel) {
            var existingUsernames = ["sebastieno"];
    
            ngModel.$asyncValidators.uniqueusername= function(value) {
                if (ngModel.$isEmpty(value)) {
                    return $q.when();
                }
              
                var def = $q.defer();
    
                $timeout(function() {
                    if (existingUsernames.indexOf(value) === -1) {
                        // Le username n'existe pas
                        def.resolve();
                    } else {
                        def.reject();
                    }
                }, 2000);
    
                return def.promise;
            };
        }
    };
});
```

## Utilisation du validateur

L'utilisation de ces directives de validation se fait de manière identique à celles déjà fournies par AngularJS. Il suffit de positionner les directives correspondantes sur les input liés à la directive `ngModel` puis d'accéder aux propriétés `$error` des champs du formulaire :

```html
<form name="form" novalidate>
    <div>
        Login:
        <input type="text" ng-model="name" name="name" unique-username-validator />
        <span ng-show="form.name.$pending.uniqueusername">Vérification de la disponibilité du login...</span>
        <span ng-show="form.name.$error.uniqueusername">Le login existe déjà</span>
    </div>
      
    <div>
        Téléphone:
        <input type="text" ng-model="phone" name="phone" phone-validator />
        <span ng-show="form.phone.$error.phone">Le numéro de téléphone est incorrect</span>
    </div>
</form>
```

Vous trouverez le code de cette directive sur le plnkr suivant : <a href="http://embed.plnkr.co/sQp9gK/preview" target="_blank">Custom validators AngularJS 1.3</a>.

Ce n'est pas plus compliqué que ça.

Bonnes validations !

