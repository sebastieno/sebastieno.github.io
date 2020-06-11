---
layout: post
title: 'Créer un composant de composition de query avec KnockoutJS'
date: 2014-09-29
categories: knockoutjs
resume: 'Créons un composant permettant de composer des requêtes (en spécifiant un champ, un opérateur et une valeur) via le framework KnockoutJS.'
tags: query-composer knockout-js
---
Récemment, j'ai eu besoin de créer un composant Web permettant de composer des requêtes, de manière identique à TFS :

![Visual Studio Online](https://sebastienollivier.blob.core.windows.net/blog/query-composer/vso.png =682x125 "Visual Studio Online")

L'idée est de pouvoir sélectionner un champ, parmi une liste, sur lequel on souhaite appliquer un filtre, puis de renseigner une valeur qui sera utilisée pour le filtre et un opérateur (pour le composant que l'on va créer, le filtre s'appliquera uniquement via l'opérateur =). On peut ensuite combiner plusieurs requêtes, via les opérateurs ET ou OU.

L'avantage de ce type de composant est de donner la possibilité à l'utilisateur de créer des requêtes personnalisées et totalement dynamiques.

On va voir dans cet article comment un tel composant peut être créé assez rapidement, en donnant le résultat suivant :

![Query Composer](https://sebastienollivier.blob.core.windows.net/blog/query-composer/querycomposer.png =914x202 "Query Composer")

## Choix des technos

Pour ce composant, on va utiliser le framework <a href="http://knockoutjs.com/" target="_blank">_KnockoutJS_</a>. On aurait pu partir sur du JavaScript classique, voir du _jQuery_, mais l'avantage d'un tel framework est d'apporter la fonctionnalité de binding, qui va nous éviter d'avoir à faire beaucoup de manipulations sur le HTML, et de permettre une meilleure structuration de notre code JavaScript. Le choix d'un autre framework MV* aurait aussi pu convenir.

En plus de KnockoutJS, on va utiliser <a href="http://www.typescriptlang.org/" target="_blank">_TypeScript_</a> qui va apporter une phase de compilation à notre code et un typage statique, permettant d'améliorer la productivité et la maintenabilité du composant.


## ViewModel KnockoutJS

### Création des modèles

Plusieurs notions vont être présentes dans ce composant. Pour chacune de ces notions, on va créer des classes TypeScript qui feront office de modèle.


**FieldType:** correspond à un type de champ. Pour l'instant, deux types de champs existent, un champ de type saisie libre et un champ de type liste de choix.


```typescript
export enum FieldTypes {
    Text,
    List
}
```

**FieldDefinition:** correspond à la définition d'un champ. Un champ est caractérisé par un type de champ, par un libellé utilisé pour l'affichage ainsi que par un nom correspondant au nom de la propriété sur laquelle se fait le filtre. Dans le cas d'un champ de type liste, la définition contient en plus la liste des valeurs possibles.

```typescript
export interface FieldDefinition {
    name: string;
    text: string;
    type: FieldTypes;
}

export class TextFieldDefinition implements FieldDefinition {
    public name: string;
    public text: string;
    public type: FieldTypes = FieldTypes.Text;

    constructor(name: string, text: string) {
        this.name = name;
        this.text = text;
        this.type = FieldTypes.Text;
    }
}

export class ListFieldDefinition implements FieldDefinition {
    public name: string;
    public text: string;
    public values: ListValue[];
    public type: FieldTypes = FieldTypes.Text;

    constructor(name: string, text: string, values: ListValue[]) {
        this.name = name;
        this.text = text;
        this.values = values;
        this.type = FieldTypes.List;
    }
}
```

**Query:** correspond à une requête. La requête est définie par un champ sur lequel s'applique le filtre ainsi qu'une valeur. La propriété operator permet de définir l'opérateur, ET ou OU, entre cette requête et la précédente.


```typescript
export class Query {
    public field: KnockoutObservable<FieldDefinition> = ko.observable(null);
    public value: KnockoutObservable<string> = ko.observable("");
    public operator: KnockoutObservable<string> = ko.observable("");
}
```

### Création du ViewModel

On va maintenant créer le ViewModel qui sera utilisé pour ce composant. Ce ViewModel sera composé d'une liste de requêtes, d'une liste de définitions de champs et des opérateurs ET et OU :

```typescript
export class QueriesViewModel {
    public queries: KnockoutObservableArray<Model.Query> = ko.observableArray([]);
    public fieldsDefinition: Model.FieldDefinition[];
    public operators: any = [{ name: 'ET', value: '&&' }, { name: 'OU', value: '||' }];
}
```

A noter que la propriété queries est de type KnockoutObservableArray puisqu'elle correspond à la liste des requêtes qui seront renseignées par l'utilisateur, et sera utilisée dans la vue via du binding.

En plus de ces propriétés, deux méthodes permettant l'ajout et la suppression d'une query sont ajoutées :

```typescript
export class QueriesViewModel {
    addQuery(): void {
        this.queries.push(new Model.Query());
    }

    removeQuery(query: Model.Query): void {
        this.queries.remove(query);
    }
}
```

On va enfin rajouter un constructeur, prenant en paramètres une liste de définitions de champs ainsi qu'une liste de requêtes :

```typescript
export class QueriesViewModel {
    constructor(fieldsDefinition: Model.FieldDefinition[], queries: any) {
        this.fieldsDefinition = fieldsDefinition;

        if (queries) {
            for (var i = 0; i < queries.length; i++) {
                [...] // Construct queries
            }
        }
    }
}
```

## Template KnockoutJS

Le ViewModel étant maintenant en place, il reste à créer le template KnockoutJS.

Le template sera composé d'une liste, contenant les requêtes, et d'un bouton permettant d'en ajouter de nouvelles.

```html
<script type="text/html" id="queryComposerTemplate">
    <ul data-bind="foreach: queries" class="queries">
        [...]
    </ul>

    <button type="button" data-bind="click: addQuery">+</button>
</script>
```
Pour chaque requête, on va afficher une liste déroulante permettant de sélectionner l'opérateur (sauf pour la première requête), une liste déroulante permettant de sélectionner le type de champ et un champ permettant de saisir la valeur du filtre.

```html
<li class="query" data-bind="css: { or: operator() === '||', and: operator() === '&&' }">
    <select class="query-field-operator" data-bind="visible: $index() > 0,
        options: $parent.operators,
        optionsText: 'name',
        value: operator,
        optionsValue: 'value'"></select>

    <select class="query-field-type" data-bind="options: $parent.fieldsDefinition,
        optionsText: 'text',
        value: field,
        optionsCaption: 'Sélectionnez un type de champ...'"></select>

    <span data-bind="if: field() && field().type == QueryComposer.Model.FieldTypes.Text">
        =
        <input class="query-field-value" type="text" data-bind="value: value" />
    </span>
    <span data-bind="if: field() && field().type == QueryComposer.Model.FieldTypes.List">
        =
        <select class="query-field-value" data-bind="options: field().values,
            optionsText: 'text',
            optionsValue: 'value',
            value: value,
            optionsCaption: 'Sélectionnez une valeur...'"></select>
    </span>

    <button class="query-btn" type="button" 
        data-bind="click: function() { $parent.removeQuery($data); }">
        x
    </button>
```

Le type de l'input permettant la saisie de la valeur va dépendre du type de champ sélectionné. S'il s'agit d'un type saisie libre, une textbox sera affichée. S'il s'agit d'un type liste, une liste déroulante, contenant les différentes valeurs définies, sera affichée.

Pour pouvoir intégrer le composant dans un formulaire HTML et poster les informations renseignées de manière classique, on va rajouter, pour chaque requête, des input hidden contenant les valeurs sélectionnées :

```html
<input type="hidden" data-bind="attr : { name: 'queries[' + $index() + '].type' }, value: field() ? field().type : ''" />
<input type="hidden" data-bind="attr : { name: 'queries[' + $index() + '].field' }, value: field() ? field().name : ''" />
<input type="hidden" data-bind="attr : { name: 'queries[' + $index() + '].value' }, value: value" />
<input type="hidden" data-bind="attr : { name: 'queries[' + $index() + '].operator' }, value: operator" />
```

## Utilisation du composant

Le composant est maintenant prêt. Pour l'utiliser, il faut dans un premier temps inclure le template dans la page :

```html
<div id="query-composer" data-bind="template : { name: 'queryComposerTemplate' }">
```

Pour l'exemple, on va définir 4 types de champs différents: un champ titre de type saisie libre, un champ état de type liste, un champ itération de type liste et un champ zone, également de type liste : 


```js
var statesList = [
    { text: "Nouveau", value: 0 },
    { text: "En cours", value: 1 },
    { text: "Fermé", value: 2 },
    { text: "Annulé", value: 3 }
];

var iterationsList = [
    { text: "Iteration 1", value: 0 },
    { text: "Iteration 2", value: 1 },
    { text: "Iteration 3", value: 2 },
    { text: "Iteration 4", value: 3 },
    { text: "Iteration 5", value: 4 },
    { text: "Iteration 6", value: 5 }
];

var areasList = [
    { text: "Frontend", value: 0 },
    { text: "Backend", value: 1 },
    { text: "Design", value: 2 }
];

var fieldsDefinition = [
    new QueryComposer.Model.TextFieldDefinition("Title", "Titre"),
    new QueryComposer.Model.ListFieldDefinition("State", "Statut", statesList),
    new QueryComposer.Model.ListFieldDefinition("Iteration", "Itération", iterationsList),
    new QueryComposer.Model.ListFieldDefinition("Area", "Zone", areasList),
];
```

Une fois la définition des champs effectuée, il reste à instancier le ViewModel et à l'appliquer sur la div en utilisant la méthode `ko.applyBindings`.

```js
var vm = new QueryComposer.QueriesViewModel(fieldsDefinition);
ko.applyBindings(vm, document.getElementById("query-composer"));
```

L'exécution de la page donne le résultat suivant (après avoir renseigné quelques queries) :

![Query Composer](https://sebastienollivier.blob.core.windows.net/blog/query-composer/querycomposer.png =914x202 "Query Composer")

## Et après ?

Vous trouverez les sources complètes du composant sur le github suivant : <a href="https://github.com/sebastieno/query-composer" target="_blank">https://github.com/sebastieno/query-composer</a>.

Il contient les scripts et le template du composant, dans le répertoire knockoutjs, ainsi qu'une application ASP.NET MVC d'exemple, dans le dossier sample. Je vous invite à le tester et à le modifier selon vos besoins, voire même à contribuer si vous le souhaitez. Et évidemment, si vous avez des questions/remarques/optimisations, n'hésitez pas.

On verra dans un prochain article comment interagir avec les requêtes sélectionnées, côté serveur en ASP.NET.

Bon requêtage !
