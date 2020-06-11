---
layout: post
title: 'Créer un Component AngularJS en TypeScript'
date: 2017-08-02
categories: javascript
resume: 'Quelle syntaxe TypeScript utiliser pour créer un component AngularJS ?'
tags: angularjs typescript component
---
Fin 2015, j'ai écrit un article expliquant <a href="http://sebastienollivier.fr/blog/javascript/angularjs-typescript" target="_blank">comment écrire du code AngularJS en utilisant TypeScript</a>. On avait vu comment déclarer un contrôleur, un filtre, une directive, etc. En reprenant un ancien projet AngularJS, je me suis rendu compte que je n'avais pas expliqué la syntaxe permettant de créer un composant (pour ma défense, cette notion n'existait pas encore et a été apportée par AngularJS 1.5).

Pour résumer très rapidement, un composant AngularJS reprend la logique et le principe des composants Angular (la nouvelle version) : créer des éléments réutilisables, composer son application en assemblant un ensemble de composants, etc.

La déclaration en TypeScript d'un composant se rapproche assez de ce que l'on a vu pour la déclaration d'une directive. Il faut commencer par créer une classe représentant le contrôleur du composant (dans l'exemple suivant un simple contrôleur permettant d'incrément ou décrémenter une propriété).

```typescript
class MoreLessComponentController {
    value: number;
    
    constructor() {  
        this.value = this.value || 0;
    }
    
    more = () => {
        this.value++;
    };
    
    less = () => {
        if (this.value > 0) {
            this.value--;
        }
    }
}
```
Ensuite il faut déclarer le composant en lui-même en créant une classe implémentant `ng.IComponentOptions` :

```typescript
class MoreLessComponent implements ng.IComponentOptions { 
    controllerAs = 'vm';
    controller = MoreLessComponentController;
    bindings = {
        value: '='
    };
    
    template: string = `<div class='more-less'>
            <button data-ng-click='vm.more()'>+</button>
            <button data-ng-disabled='vm.value <= 0' data-ng-click='vm.less()'>-</button>
        </div>
    `;
    
    static factory() {
        var component = () => {
            return new MoreLessComponent();
        };
        
        return component;
    }
}

angular.module('easypass').component('moreLess', MoreLessComponent.factory());

```
Le composant déclare une factory permettant de renvoyer une nouvelle instance, afin d'être enregistré dans le module Angular. La propriété `bindings` permet de faire le mapping entre les propriétés du contrôleur et celles exposées par le composant. A noter également que la relation entre le composant et son contrôleur se fait en renseigner le type du contrôleur via la propriété `controller`.

Et voilà, vous pouvez créer vos composants AngularJS assez facilement en TypeScript.

Bon composants !


