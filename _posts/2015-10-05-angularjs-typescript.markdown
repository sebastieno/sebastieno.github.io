---
layout: post
title: 'AngularJS et TypeScript'
date: 2015-10-05
categories: javascript
resume: 'Découvrons dans cet article toutes les syntaxes TypeScript permettant de créer des éléments AngularJS (controllers, directives, filtres, services, etc.).'
tags: angularjs typescript
---
Lorsque l'on créé des applications avec AngularJS, on se rend vite compte que la dynamicité du langage JavaScript (typage dynamique et langage interprété) rend la maintenabilité du code difficile (notamment les phases de refactoring qui se révèlent très compliquées), et ce malgré les patterns apportés par AngularJS (MVC/MVVM, IoC).

TypeScript résout ce problème en apportant une phase de compilation et un typage statique qui va permettre d'améliorer grandement la maintenabilité de vos applications (et votre productivité par la même occasion). Si vous avez l'habitude d'utiliser AngularJS en JavaScript, le pas pour passer à TypeScript peut paraître difficile à franchir. En JavaScript, on fournit des fonctions `factory` à AngularJS alors qu'en TypeScript, on travaille avec des classes.

On va voir dans cet article comment chaque type d'élément AngularJS se déclare en TypeScript.

## Installation des d.ts

Avant de commencer à développer son application AngularJS en TypeScript, il est nécessaire de récupérer les interfaces TypeScript, fichiers d.ts, décrivant les différentes méthodes du framework AngularJS. Vous pouvez soit les télécharger <a href="https://github.com/borisyankov/DefinitelyTyped/tree/master/angularjs" target="_blank">ici</a> soit utiliser <a href="https://github.com/Definitelytyped/tsd#readme" target="_blank">tsd</a>, paquet npm, qui le fera pour vous.

Si vous passez par tsd, il vous faudra exécuter les deux commandes suivantes, permettant respectivement d'installer le paquet npm en global et de télécharger les fichiers d.ts d'AngularJS dans un répertoire typings :

```javascript
npm install tsd -g
tsd query -rosa install angularjs
```

## Controller

Il existe deux méthodes permettant de fournir des données à la vue depuis un contrôleur. On va voir quelle est la syntaxe à adopter pour chacune de ces méthodes. 

### Controller avec scope

La méthode classique consiste à enrichir un objet `$scope`. En TypeScript, il faut donc commencer par créer une interface modélisant cet objet et étendant l'interface `ng.IScope` (pour accéder à toutes les méthodes du scope, `$watch`, `$on`, etc.).

```javascript
interface IHomeControllerScope extends ng.IScope {
	isLoading: boolean;
	items: any[];
	refreshItems: () => void;
}
```

Ensuite, il suffit de créer une classe représentant le contrôleur.

```js
module DemoJsToTs {
	class HomeController {
		constructor(private $scope: IHomeControllerScope , private itemsService: ItemsService) {
			this.$scope.refreshItems = () => {
				this.$scope.isLoading = false;
				this.itemsService.getItems().then((items) => {
					this.$scope.items = items;
				}).finally(() => this.$scope.isLoading = true);
			}
			
			this.$scope.refreshItems();
		}
	}

	angular.module("demoJsToTs").controller("homeController", ["$scope", "itemsService", HomeController]);
}
```

Cette classe prend dans son constructeur le scope (du type de l'interface que l'on vient de créer) et le service `itemsService` dont il dépend, qui seront fourni par le mécanisme d'injection de dépendances d'AngularJS. Toutes les données du scope sont ensuite initialisées dans le constructeur.

### ControllerAs
La syntaxe `controllerAs` consiste à enrichir directement le contrôleur, sans passer par un objet `$scope` intermédiaire. Dans ce cas en TypeScript, on n'aura pas besoin de définir une interface pour le scope, et on pourra directement ajouter nos propriétés et méthodes sur la classe du contrôleur.

Le même contrôleur se déclarerait maintenant de la façon suivante :
```js
module DemoJsToTs {
	class HomeController {
		constructor(private itemsService: ItemsService) {
			this.refreshItems();
		}
		
		public isLoading: boolean;
		public items: any[];
		
		public refreshItems() {
			this.isLoading = true;
			this.itemsService.getItems().then((items) => {
				this.items = items;
			}).finally(() => this.isLoading = true);
		}
	}
	
	angular.module("demoJsToTs").controller("homeController", ["itemsService", HomeController]);
}
```

Si vous avez besoin d'utiliser des fonctions exposées par l'objet `$scope`, il faudra en plus injecter le scope dans le constructeur du contrôleur.

## Service

Un service se déclare simplement en créant une classe TypeScript : 

```js
module DemoJsToTs {
	export class ItemsService {
		constructor(private $http: ng.IHttpService, private $q: ng.IQService) {
			
		}
		
		getItems() : ng.IPromise<any[]> {
			var deferred = this.$q.defer();
			
			this.$http({
				url: 'http://...',
				method: 'GET'
			}).success(result => {
				deferred.resolve(result);
			}).error(e => {
				deferred.reject(e);
			});
			
			return deferred.promise;
		}
	}
	
	angular.module("demoJsToTs").service("itemsService", ["$http", "$q", ItemsService]);
}
```

Notez que l'on exporte le service (via le mot clef `export`) pour que l'on puisse le référencer dans d'autres classes.

Volontairement, je n'illustre pas les `factory` d'AngularJS (qui sont uniquement une alternative syntaxique aux services) puisqu'elles ne s'intègrent pas très bien au typage apporté par TypeScript.

## Provider

Pour définir un provider AngularJS en TypeScript, il faut d'abord créer le service qui sera fourni par le provider, comme vu précédemment.

```js
module DemoJsToTs {
	export class AuthenticationService {
		constructor(private $http: ng.IHttpService, private clientId: string, private clientSecret: string) {
		}
		
		renewToken() {
			[...]
		}
	}
}
```

Ensuite, on va créer une classe implémentant l'interface `IServiceProvider` et fournissant une méthode `$get`, qui retourne une nouvelle instance du service.

```js
module DemoJsToTs {
	export class AuthenticationServiceProvider implements ng.IServiceProvider {
		private clientId: string = null;
		private clientSecret: string = null;

		configure(clientId: string, clientSecret: string) {
			this.clientId = clientId;
			this.clientSecret = clientSecret;
		}

		$get = ["$http", function ($http: ng.IHttpService) {
			return new AuthenticationService($http, this.clientId, this.clientSecret);
		}];
	}
	
	angular.module("demoJsToTs").provider("authenticationService", [AuthenticationServiceProvider]);
}
```

Notez ici que l'injection des dépendances du service ne se fait pas lors de l'enregistrement du provider dans AngularJS mais lors de l'appel à la méthode `$get`.

Dans l'exemple précédent, le provider expose une méthode `configure` qui permet d'initialiser les paramètres `clientId` et `clientSecret` dans la phase `config` du module, avant l'exécution de l'application.

## Filter

Pour les filtres, contrairement aux autres éléments AngularJS, TypeScript n'apporte pas grand-chose. La syntaxe ressemble donc fortement à celle que l'on utiliserait en JavaScript.

```js
module DemoJsToTs {
	angular.module('demoJsToTs').filter('myFilterDoNothingWoops', () => {
		return (data: string) => {
			return data;
		}
	});
}
```

## Directive
La déclaration d'une directive est un peu plus subtile que ce que l'on a vu jusque-là. AngularJS attend qu'on lui fournisse une méthode `factory` créant une instance de la directive, et non pas une classe comme pour les contrôleurs ou les services.

L'idée est donc de créer une classe implémentant l'interface `IDirective` puis d'y déclarer une méthode statique servant de factory. C'est ensuite cette méthode qui sera enregistrée dans AngularJS.
```js
module DemoJsToTs {
	interface IItemsDirectiveScope extends ng.IScope {
		items: any[];
	}
	
	class ItemsDirective implements ng.IDirective {
		constructor(private itemsService: ItemsService) {}
		
		scope = {
			items: '='
		}
		
		link = (scope: IItemsDirectiveScope , element: ng.IAugmentedJQuery, attrs: ng.IAttributes) => {
		}

		public static Factory() {
			var directive = (itemsService: ItemsService) => {
				return new ItemsDirective (itemsService);
			}
			
			return directive;
		}
	}
	
	angular.module('demoJsToTs').directive("items", ["itemsService", ItemsDirective .Factory()]);
}
```
Nous avons fait le tour des différentes syntaxes à utiliser pour déclarer des éléments AngularJS en TypeScript. TypeScript apporte vraiment beaucoup en stabilité et en confort de développement et je ne peux que vous conseiller de l'utiliser pour vos prochains projets AngularJS (et de manière générale dès que vous avez beaucoup de JavaScript).

A bientôt !

