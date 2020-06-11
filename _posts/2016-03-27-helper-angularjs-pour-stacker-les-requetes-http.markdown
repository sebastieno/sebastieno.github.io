---
layout: post
title: 'Helper AngularJS pour stacker les requêtes HTTP'
date: 2016-03-27
categories: angularjs
resume: 'Voici un helper AngularJS, sous la forme d''un service, permettant de s''assurer qu''une requête n''est effectuée qu''une seule fois simultanément, même si on l''appelle plusieurs fois.'
tags: angularjs
---
Lorsque l'on encapsule un appel HTTP dans une méthode, on souhaite souvent pouvoir s'assurer que cette requête n'est effectuée qu'une seule fois simultanément, même si l'on appelle plusieurs fois la méthode.

La classe Typescript suivante permet de s'assurer de cela.
```javascript
module IS {
    export class HttpRequestLocker {
        constructor(private $http: ng.IHttpService,
            private $q: ng.IQService,
            private requestConfig: ng.IRequestConfig,
            private successCallback?: (data : any) => any) {
        }

        private waitingDeferred: ng.IDeferred<any>[];

        execute(): ng.IPromise<any> {
            var deferred = this.$q.defer();

            if (!this.waitingDeferred || this.waitingDeferred.length === 0) {
                this.waitingDeferred = [];
                this.$http(this.requestConfig).success((result) => {
                    if (this.successCallback) {
                        result = this.successCallback(result);
                    }

                    for (var i = 0; i < this.waitingDeferred.length; i++) {
                        this.waitingDeferred[i].resolve(result);
                    }
                }).error((error) => {
                    for (var i = 0; i < this.waitingDeferred.length; i++) {
                        this.waitingDeferred[i].reject(error);
                    }
                }).finally(() => {
                    this.waitingDeferred = null;
                });
            }

            this.waitingDeferred.push(deferred);

            return deferred.promise;
        }
    }
}
```

Vous pouvez également trouver la version Javascript ici : <a href="https://gist.github.com/sebastieno/fb5d305f89acd0faec8a" target="_blank">https://gist.github.com/sebastieno/af3404d62ecbc11c2150</a>.

Le principe consiste à stacker les `deferred` (en déclenchant ou non un appel HTTP), puis à toutes les résoudre une fois que l'unique requête est terminée.

L'utilisation de cet helper est plutôt simple. Il suffit dans un premier temps d'instancier un nouvel objet `HttpRequestLocker`, en lui passant les services `$http` et `$q` ainsi qu'une configuration HTTP :
```javascript
constructor($http: ng.IHttpService, $q: ng.IQService) {
	this.httpRequestLocker = new IS.HttpRequestLocker(this.$http,
		this.$q,
		{
			method: 'GET',
			url: 'http://localhost/oauth/me'
		});
}
```

Ensuite, pour lancer la requête HTTP, il faut appeler la méthode `execute` qui renverra une promise. S'il y a déjà une requête en cours, l'appel ne sera pas effectué et la promise sera résolue une fois la requête en cours terminée.

```javascript
getMyData() : ng.IPromise<any> {
	this.httpRequestLocker.execute()
		.then((data) => {
		}, (error) => {
		});
}
```
Il est également possible d'ajouter un quatrième paramètre pour effectuer un traitement sur le retour de la requête avant de résoudre toutes les promises en attente (par exemple pour stocker les données dans le `localStorage`) :

```javascript
this.httpRequestLocker = new IS.HttpRequestLocker(this.$http,
	this.$q,
	{
		method: 'GET',
		url: 'http://localhost/oauth/me'
	}, 
	(data) => {
		localStorage.setItem('data', data);
		return data;
	});
```
Cet helper a été créé pour AngularJS mais peut très bien être adapté à tous les Frameworks proposant un mécanisme de requêtage HTTP et de promises.

Bon requêtage !

