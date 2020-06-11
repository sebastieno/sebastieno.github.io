---
layout: post
title: 'Stratégies de lazy loading de modules avec Angular'
date: 2018-03-07
categories: angular
resume: 'Quelles sont les différentes possibilités permettant de déclencher le chargement d''un module Angular lazy loadé ?'
tags: angular lazyloading modules
---
Le lazy loading de modules est une fonctionnalité d'Angular permettant de charger du code applicatif lorsque celui-ci sera sollicité, plutôt que de charger toute l'application dès son démarrage. L'idée est qu'Angular, à la compilation via la cli et WebPack, va découper l'application en plusieurs fichiers ou _chunk_ :

![Angular chunks](https://sebastienollivier.blob.core.windows.net/blog/strategies-lazyloading-module-angular/chunks.jpg =654x144 "Angular chunks")

L'avantage de ce mécanisme se situe évidemment au niveau des performances, puisque l'on va pouvoir proposer un affichage de l'application beaucoup plus rapidement en ne chargeant que la partie nécessaire, et en déférrant le chargement des autres parties.

_Pour plus de détails sur le lazy loading de module avec Angular, je vous laisse regarder la présentation effectuée par Daniel D. et William K au meetup ngParis : <a href="https://www.slideshare.net/DanielDjordjevic/meetup-angular-paris-feature-modules" target="_blank">Meetup Angular Paris - Feature Modules</a>._

Angular propose plusieurs stratégies définissant le moment où seront chargés les modules lazy loadés. Voyons ensemble ces stratégies.

## Chargement à la demande

Par défaut, un module lazy loadé sera chargé lorsque sa route sera appelée.

```typescript
export const routes = [{ path: 'admin', loadChildren: './modules/admin/admin.module#AdminModule' }];
```

Dans l'exemple précédent, le module _AdminModule_ est lié à la route _admin_. Une fois que l'utilisateur naviguera vers cette route, Angular chargera le chunk du module correspondant.

Cette stratégie permet de différer au maximum le chargement d'un module mais a pour conséquence de provoquer une légère latence lors de la navigation, puisqu'Angular devra attendre la fin du chargement du chunk avant de pouvoir instancier le composant ciblé.

## Preloading de tous les modules

Angular fournit une alternative au comportement précédent via la classe _PreloadAllModules_. Cette stratégie permet de charger tous les modules sans attendre qu'ils soient _visités_. Pour l'utiliser, il suffit de fournir la classe _PreloadAllModules_ en paramètre de la méthode _forRoot_ du module _RouterModule_ :

```typescript
RouterModule.forRoot(ROUTES, { preloadingStrategy: PreloadAllModules })
```

L'avantage évident ici est que l'on va dans un premier temps charger uniquement la partie nécessaire au démarrage de l'application, puis les autres modules seront chargés en background. On gagne donc en performance au premier affichage sans ajouter un surcout à la navigation dans un nouveau module.

Le côté négatif est que l'on va beaucoup solliciter le réseau au démarrage de l'application, ce qui pourrait entraîner des problématiques en fonction du contexte d'exécution (device bas de game, etc.).

## Preloading de certains modules

Il est également possible de créer sa propre stratégie de chargement. Le besoin le plus fréquent est de marquer certaines routes comme nécessitant un pré-chargement.

```typescript
{ path: 'moduleA', loadChildren: './modules/moduleA/module-a.module#ModuleA', data: { preload: true } },
{ path: 'moduleB', loadChildren: './modules/moduleB/module-b.module#ModuleB' }
```

Dans la déclaration des routes précédentes, le module A est marqué en preload, le module B ne l'est pas. On peut ensuite créer une classe implémentant _PreloadingStrategy_ comme ci-après :

```typescript
export class PreloadTaggedModuleStrategy implements PreloadingStrategy {
    preload(route: Route, load: Function): Observable<any> {
       	return route.data && route.data.preload ? load() : of(null);
    }
}
```

L'implémentation vérifie simplement si la route possède la propriété _preload_ à _true_, et si c'est le cas appele la fonction _load_ qui déclenchera le chargement du module. L'enregistrement de la stratégie se fait toujours via la méthode _forRoot_ du _RouterModule_ :

```typescript
RouterModule.forRoot(ROUTES, { preloadingStrategy: PreloadTaggedModuleStrategy })
```

On obtient ici un comportement hybride, puisque l'on va pre-charger uniquement les routes qui nous intéresse. On gagne donc en temps d'affichage sans pour autant surcharger le réseau.

## Stratégie custom

Angular met également à disposition la classe _NgModuleFactoryLoader_ permettant de déclencher le chargement d'un module sans avoir à passer par une _PreloadingStrategy_. Cela est extrêmement utile lorsque l'on souhaite créer ses propres scénarios, adaptés à son application.

On peut par exemple imaginer une directive qui, au clic sur un élément, déclenchera le chargement d'un module :

```typescript
@Directive({ 
    selector: '[loadModuleOnClick]'
})
export class LoadModuleOnClickDirective {
    @Input('loadModuleOnClick') module: string;
    
    constructor(private element: ElementRef, private loader: NgModuleFactoryLoader, renderer: Renderer) {
        const unsubscribe = renderer.listen(element.nativeElement, 'click', (evt) => {
            this.loader.load(this.module);
            unsubscribe();
        });
    }
}
```

L'utilisation se fait alors via la syntaxe suivante :

```html
<div loadModuleOnClick='./modules/modulea/module-a.module#ModuleA'>[…]</div>
```

On peut également imaginer un scénario où l'on chargerait un module d'administration à la connexion d'un utilisateur uniquement si celui-ci possède le rôle _Administrateur_.

Toutes ces possibilités nous permettent d'adapter le chargement de modules à nos besoins afin de proposer la meilleure expérience à l'utilisateur.

Bon chargements !
