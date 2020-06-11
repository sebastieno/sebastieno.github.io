---
layout: post
title: 'Utiliser les merges Cordova dans une application AngularJS avec TypeScript'
date: 2015-11-30
categories: cordova
resume: 'Comment organiser son application afin de faire cohabiter une application AngularJS, utilisant TypeScript, et le mécanisme de ''merges'' d''une application Cordova ?'
tags: cordova merge angularjs typescript
---
Le principe du dossier `merge` de Cordova est de pouvoir injecter du code sur une plateforme spécifique (iOS, Android, Windows, etc.). A la compilation de l'application, Cordova embarquera les éléments du dossier correspondant à la plateforme cible.

![Dossier merge](https://sebastienollivier.blob.core.windows.net/blog/merge-cordova-angular/arbo.jpg =369x249 "Dossier merge")

Dans le cadre d'une application AngularJS, le dossier _merge_ peut être utilisé pour déclarer des implémentations différentes par plateforme d'un même service. C'est ce scénario que nous avons mis en place pour une application dans laquelle il était nécessaire d'afficher une carte GoogleMap. Sur Android et iOS, la solution technique consiste à injecter le script de GoogleMap directement dans l'application, de manière classique. Par contre sur Windows, l'injection de scripts étant bloquée pour des questions de sécurité, il est nécessaire d'utiliser une ms-web-view dans laquelle se fera l'injection. _Pour plus d'informations sur l'implémentation technique, vous pouvez vous référer au lien suivant <a href="https://blogs.infinitesquare.com/posts/mobile/implementer-une-googlemap-dans-une-application-cordova-avec-angularjs-et-typescript" target="_blank">https://blogs.infinitesquare.com/posts/mobile/implementer-une-googlemap-dans-une-application-cordova-avec-angularjs-et-typescript</a>.

Dans cet article, je vais vous présenter comment nous avons dû organiser l'application, et notamment les builds TypeScript avec Gulp, pour faire cohabiter ces deux solutions différentes.

## Structure

L'idée de notre solution est de garder le maximum de code commun à toutes les plateformes. La différence technique entre Android / iOS et Windows tient dans la façon d'initialiser la carte GoogleMap, et non dans la façon d'interagir avec.

La structure mise en place est la suivante :

![Structure de la solution](https://sebastienollivier.blob.core.windows.net/blog/merge-cordova-angular/structuresolution.jpg =800x450 "Structure de la solution")

Nous avons une directive `map` permettant d'afficher une carte GoogleMap. Cette directive se base sur un service `mapHandler` qui est chargé d'instancier la carte, en injectant le script ou en passant par une ms-web-view. Et nous avons également un deuxième service, `mapSupervisor`, utilisé par le `mapHandler`, dont le rôle est de piloter une instance d'une carte GoogleMap (changement de position, ajout de points, etc.).

Dans cette solution, c'est le service `mapHandler` qui se retrouve dans le dossier `merge`, afin de pouvoir fournir une implémentation différente par plateforme.

## Quel est le problème ?

La compilation de l'application doit se faire en deux parties. L'application en elle-même doit être compilée, sans le contenu du dossier `merge`. Ensuite, le contenu du dossier `merge` doit également être compilé, sans le reste de l'application.

Le problème est lié à la dépendance circulaire entre l'application et le dossier `merge`. La directive `map` (dans l'application) a besoin du service `mapHandler` (dans le dossier `merge`), et le service `mapHandler` est dépendant du service `mapSupervisor` (dans l'application).

##Compilation de l'application

Pour réussir à compiler l'application, sans le contenu du dossier `merge`, il faut simplement créer une interface `imapHandler`, qui se situera côté application et pas côté merge, dont hériteront toutes les implémentations des `mapHandler`. La directive `map` sera alors dépendante de cette interface et la compilation pourra se faire sans problème :

```javascript
gulp.task("ts", function () {
    gulp.src([config.typescriptFiles,
       config.typingsFiles])
        .pipe(ts({
            target: 'ES5',
            declarationFiles: false,
            noExternalResolve: true
        }))
        .pipe(gulp.dest(config.scriptsDestPath));
});
```
## Compilation des merges
La compilation du dossier `merge` est un peu plus compliquée.

Le premier problème vient du faire qu'on a plusieurs classes sous le nom de `mapHandler`, ce qui génèrera une erreur de compilation. Pour résoudre cela, il suffit de nommer différemment chaque classe de chaque plateforme (`mapHandlerForIOS`, `mapHandlerForAndroid` et `mapHandlerForWindows` par exemple) mais de les inscrire dans AngularJS en tant que `mapHandler`:
```javascript
export class MapHandlerForAndroid implements IMapHandler {}

angular.module("myModule").service("mapHandler", [MapHandlerForAndroid]);
```
```javascript
export class MapHandlerForWindows implements IMapHandler {}

angular.module("myModule").service("mapHandler", [MapHandlerForWindows]);
```
```javascript
export class MapHandlerForiOS implements IMapHandler {}

angular.module("myModule").service("mapHandler", [MapHandlerForiOS]);
```

Le second problème est lié à la dépendance circulaire. Plutôt que de créer un ensemble d'interface (qui ne fonctionnerait pas si on a besoin de référencer des enum par exemple), nous allons créer un fichier de définition `d.ts` qui décriera tous les éléments de l'application utilisés par le service `mapHandler`:

```javascript
declare module MyModule {
    export interface IMapHandler {
        configure(element: string, mapConfiguration: IMapConfiguration);
        setMapFeatures(mapFeaturesContainers: IMapFeaturesContainer[]);
    }

    export class MapSupervisorService {       
        initMap(element: string, mapConfiguration: IMapConfiguration);
        refreshMap(mapConfiguration: IMapConfiguration);
        removeMapFeatures();
        showMapFeaturesCollection(mapFeaturesContainers: IMapFeaturesContainer[])
    }
    
    export interface IMapFeaturesContainer {
        mapFeatures: any;
        type: any;
    }
    
    export interface IMapConfiguration {
        mapType: any;
        zoom: any;
        position: any;
    }
}
```

Tous les éléments de l'application utilisés par `mapHandler` sont déclarés, même si par facilité ils sont définis à `any`. La compilation peut donc se faire :

```javascript
gulp.task("build-merges", function () {
    gulp.src([config.mergesFiles,
       config.typingsFiles])
       .pipe(ts({
           target: 'ES5',
           declarationFiles: false,
           noExternalResolve: false
       }))
       .pipe(gulp.dest(config.mergesDestPath));
});
```

Par contre, la compilation de l'application ne va plus fonctionner. L'ensemble des éléments déclarés dans le fichier `d.ts` le sera déjà dans l'application, ce qui fera échouer la compilation. Pour corriger ça, il faut exclure ce fichier de la build :

```javascript
gulp.task("ts", function () {
    gulp.src([config.typescriptFiles,
       config.typingsFiles,
       '!' + config.typingsSourcePath + '/mymodule.d.ts'])
        .pipe(ts({
            target: 'ES5',
            declarationFiles: false,
            noExternalResolve: true
        }))
        .pipe(gulp.dest(config.scriptsDestPath));
});
```

Et voilà, ca y est, la compilation des deux parties de l'application fonctionne et on peut proposer une implémentation du service `mapHandler` par plateforme.

Bon merges !

