---
layout: post
title: 'Mettre en cache le résultat d''un Observable via un décorateur TypeScript'
date: 2018-07-17
categories: javascript
resume: 'Voici un décorateur TypeScript permettant de mettre en cache le résultat d''un Observable.'
tags: decorator rxjs observable cache replaysubject
---
Dans l'article précédent <a href="https://sebastienollivier.fr/blog/javascript/reutiliser-observable-via-decorateur-typescript" target="_blank">Réutiliser une Observable via un décorateur TypeScript</a>, nous avons vu comment créer un décorateur TypeScript permettant de réutiliser un Observable non finalisé, dans le but par exemple d'empêcher d'effectuer plusieurs fois les mêmes requêtes HTTP en parallèle.

Nous allons voir ici comment pousser le mécanisme un peu plus loin, toujours à l'aide d'un décorateur TypeScript. L'idée est de pouvoir garder le résultat d'un Observable en cache afin de le récupérer à chaque fois que l'on appelera une méthode (sans redéclencher une requête HTTP par exemple).

# Mettre en cache un Observable

Dans un premier temps, on va se créer une fonction permettant de mettre en cache le résultat d'un Observable. Pour cela, on va utiliser un <a href="https://github.com/ReactiveX/rxjs/blob/master/doc/subject.md#replaysubject" target="_blank">ReplaySubject</a> avec un buffer de 1 afin de toujours renvoyer la dernière valeur :

```typescript
import { Observable, ReplaySubject } from 'rxjs';

export function cacheable<T>(observable: Observable<T>): Observable<T> {
  const replaySubject = new ReplaySubject<T>(1);

  observable.subscribe(
    x => replaySubject.next(x),
    x => replaySubject.error(x), 
    () => replaySubject.complete()
  );

  return replaySubject.asObservable();
}
```

# Le décorateur

Voici maintenant le code du décorateur :

```typescript
import { Observable } from 'rxjs';
import { cacheable } from '../utils';

export function Cacheable() {
  return function(
    target: any,
    propertyKey: string,
    descriptor: TypedPropertyDescriptor<(...args: any[]) => Observable<any>>
  ) {
    const originalMethod = descriptor.value;
    const cacheablePropertyName = `__${propertyKey}__cacheablecontent`;

    target[cacheablePropertyName] = {};

    descriptor.value = function(...args: any[]) {
      const valuesKey = args
        .filter(a => {
          if (typeof a === 'object') {
            console.warn('Object parameters are not supported. It has been ignored !');
            return false;
          }

          return true;
        })
        .toString();

      if (!this[cacheablePropertyName][valuesKey]) {
        this[cacheablePropertyName][valuesKey] = cacheable(originalMethod.apply(this, args));
      }

      return this[cacheablePropertyName][valuesKey];
    };
  };
}

```

Un objet nommé _\_\_&lt;nom de la méthode&gt;\_\_cacheablecontent_ est créée sur le prototype de la classe afin de contenir les observables retournés. La méthode décorée est ensuite surchargée afin de générer une clef en concaténant ses paramètres. Si un Observable a déjà été mis en cache pour ces paramètres, on le retourne, sinon on appelle la méthode d'origine en n'oubliant pas de mettre en cache son résultat.

Son utilisation se fait de la manière suivante :

```typescript
@Cacheable()
getEntryPoints(id: string): Observable<Category[]> {
  [...]
}
```

Le résultat de notre méthode est en cache, et seul le premier appel (par paramètre) déclenchera le réel traitement (par exemple le requêtage HTTP).

# Invalider le cache par clef

Ce décorateur nous permet de garder les données en cache mais à l'inconvénient d'être extrêmement statique. Une fois que l'on a appelée la méthode avec des paramètres donnés, impossible de la réexécuter une seconde fois.

Cette impossibilité peut s'avérer contraignante dans certains cas. Si par exemple notre application est multilingue, et que les données renvoyées par l'API sont dépendantes de la langue de l'utilisateur, on aimerait pouvoir invalider le cache du décorateur lorsque l'utilisateur change de langue. Pour satisfaire ce scénario, on va créer un décorateur de propriété dont le rôle sera d'identifier sur quelle clef le cache doit s'invalider :

```typescript
const CACHEABLE_KEY = '__cacheable_decorator_cacheable_key__';

export function CacheableVaryByKey() {
  return function(target: Object, propertyKey: string | symbol) {
    target[CACHEABLE_KEY] = propertyKey;
  };
}
```

On modifie ensuite notre premier décorateur pour prendre en compte cette nouvelle information :

```typescript
import { Observable } from 'rxjs';
import { cacheable } from './cacheable';

const CACHEABLE_KEY = '__cacheable_decorator_cacheable_key__';

export function Cacheable() {
  return function(
    target: any,
    propertyKey: string,
    descriptor: TypedPropertyDescriptor<(...args: any[]) => Observable<any>>
  ) {
    const originalMethod = descriptor.value;
    const cacheablePropertyName = `__${propertyKey}__cacheablecontent`;
    const cacheableKeyPropertyName = `__${propertyKey}__cacheablecontent__key`;

    target[cacheablePropertyName] = {};
    target[cacheableKeyPropertyName] = {};

    descriptor.value = function(...args: any[]) {
      const targetKey = this[CACHEABLE_KEY] ? this[this[CACHEABLE_KEY]] : undefined;

      const valuesKey = args
        .filter(a => {
          if (typeof a === 'object') {
            console.warn('Object parameters are not supported. It has been ignored !');
            return false;
          }

          return true;
        })
        .toString();

      if (!this[cacheablePropertyName][valuesKey] || this[cacheableKeyPropertyName][valuesKey] !== targetKey) {
        this[cacheablePropertyName][valuesKey] = cacheable(originalMethod.apply(this, args));
        this[cacheableKeyPropertyName][valuesKey] = targetKey;
      }

      return this[cacheablePropertyName][valuesKey];
    };
  };
}
```

La première modification est la définition d'une propriété nommée _\_\_&lt;nom de la méthode&gt;\_\_cacheablecontent\_\_key_ dont le rôle sera de stocker la valeur de la clef au moment de l'exécution de la méthode.

En plus de vérifier s'il y a un observable en cache, on va maintenant également vérifier si la valeur de la clef a changé entre l'appel de mise en cache et l'appel actuel (`this[cacheableKeyPropertyName][valuesKey] !== targetKey`). Si la valeur n'a pas changé, on renvoie l'observable en cache, sinon on appele la méthode d'origine, puis on met en cache l'observable et la clef utilisée.

L'utilisation de ces décorateurs se fait de la manière suivante :

```typescript
@CacheableVaryByKey()
userLanguage: string;

constructor(private httpClient: HttpClient, private i18nService: I18nService) {
    this.i18nService.languageChanged$.subscribe(lang => (this.userLanguage = lang));
}

@Cacheable()
getEntryPoints(): Observable<Category[]> {
  [...]
}
```

Le résultat de la méthode `getEntryPoints` est mis en cache et la propriété `userLanguage` permet d'invalider les observables en cache. De cette manière, à chaque fois que l'utilisateur change de langue, la méthode pourra être réexécutée.

Bonnes mises en cache !
