---
layout: post
title: 'Réutiliser une Observable via un décorateur TypeScript'
date: 2018-06-04
categories: javascript
resume: 'Voici un décorateur TypeScript permettant de réutiliser un Observable non finalisé.'
tags: decorator rxjs observable
---
Il est assez courant d'avoir besoin de s'assurer qu'une requête HTTP ne puisse pas être effectuée plusieurs fois en parallèle.

_Prenons par exemple le scénario d'expiration d'un jeton OAuth. Je navigue sur la page de détail d'un produit, cette page provoquant trois appels HTTP (une pour les informations du produit, une pour les commentaires et une dernière pour des suggestions d'autres produits). Au moment de cette navigation, ma session expire. Ces trois requêtes vont générer des 401 (Unauthorized) qui déclencheront un processus de refresh de jeton (via un interceptor Angular par exemple). Le problème ici est que si je ne prévois pas un mécanisme de blocage d'appels, trois requêtes HTTP de refresh de jeton seront effectués (donc en fonction de l'implémentation du refresh, je peux me retrouver avec des erreurs en plus de la surcharge réseau)._

Cette problématique n'est pas très compliquée à régler lorsque l'on utilise des <a href="http://reactivex.io/rxjs/" target="_blank">_Observables_</a>, le seul problème est qu'elle rajoute du code technique un peu lourd à l'intérieur du code métier. Pour régler cela, j'ai créé un décorateur que j'utilise assez fréquemment sur mes projets et que je vous partage aujourd'hui.

_Si vous souhaitez avoir plus d'informations sur les décorateurs (syntaxe, cas d'usages, etc.), vous pouvez vous réferrer à un de mes précédents articles : <a href="https://sebastienollivier.fr/blog/javascript/a-la-decouverte-des-decorateurs-typescript">A la découverte des décorateurs TypeScript</a>._

## Le décorateur

Voici <a href="https://gist.github.com/sebastieno/68296db2d3a3517c56ad783765cb9ecc" target="_blank">le code</a> du décorateur :

```javascript
import { Observable } from 'rxjs;
import { finalize, share } from 'rxjs/operators';

export function ReusePendingObservable() {
    return function (target: any, propertyKey: string, descriptor: TypedPropertyDescriptor<any>) {
        const originalMethod = descriptor.value;
        const pendingObservablePropertyName = `__pendingObservable${propertyKey}`;
        
        target[pendingObservablePropertyName] = null;
        descriptor.value = function (...args: any[]) {
            if (!target[pendingObservablePropertyName]) {
                const result: Observable<any> = originalMethod.apply(this, args);
                target[pendingObservablePropertyName] = result.pipe(finalize(() => target[pendingObservablePropertyName] = null), share());
            }

            return target[pendingObservablePropertyName];
        };
    };
}
```

L'idée est relativement simple. Une propriété nommée ___pendingObservable&lt;nom de la méthode&gt;_ est créée sur le prototype de la classe afin de contenir l'observable courante. La méthode décorée est ensuite surchargée afin de stocker l'observable créée dans la variable précédente. On en profite également pour nettoyer cette variable lorsque l'observable est _complete_ (via l'opérateur _finalize_) et appliquer un _share_ sur l'observable pour éviter que chaque souscription déclenche un appel HTTP.

Son utilisation se fait de la manière suivante :

```javascript
@ReusePendingObservable()
renewToken(refresh: string) : Observable<Token> {
	return this.httpClient.post([…]);
}
```

Il devient alors impossible d'effectuer plusieurs appels HTTP simultanées en appelant cette méthode et le code n'est pas pollué par toutes ces considérations techniques.

Bon requêtages !
