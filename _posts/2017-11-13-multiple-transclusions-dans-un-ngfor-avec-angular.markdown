---
layout: post
title: 'Multiple transclusions dans un ngFor avec Angular'
date: 2017-11-13
categories: angular
resume: 'Comment peut-on profiter du mécanisme de transclusions, permettant d''injecter un template à l''intérieur d''un composant, depuis un ngFor ?'
tags: angular ngfor ngtemplate transclusion let
---
Le principe de transclusion permet d'injecter un template à l'intérieur d'un composant, rendant ce dernier hautement générique. C'est par exemple très utile lorsque l'on veut créer un composant pour gérer les popin/modal dont le contenu est dynamique et dépendant du contexte d'utilisation, et donc du composant parent.

Mais ce mécanisme ne fonctionne pas lorsqu'on l'utilise à l'intérieur d'un *ngFor. Voyons ensemble comment contourner cela.

## Cas d'une transclusion simple
Le composant suivant utilise la transclusion, grâce au tag `ng-content` déclaré dans son template :

```typescript
import { Component } from '@angular/core';
@Component({
    selector: 'hello',
    template: `Hello <ng-content></ng-content> !`
})
export class HelloComponent {
}
```
L'utilisation du composant se fait de la façon suivante :

```html
<hello>Sébastien</hello>
```
Le label "Sébastien" correspond à l'élément transclut, c'est-à-dire à l'élément qui sera injecté dans le composant `HelloComponent` à l'emplacement de `ng-content`. La vue résultante sera alors :

```html
Hello Sébastien !
```
## Cas d'une transclusion dans un *ngFor
Changeons le composant pour prendre en entrée un nombre de messages à afficher :

```typescript
@Component({
    selector: 'multiple-hello',
    template: `<ng-container *ngFor='let item of items'>
                  Multiple Hello <ng-content></ng-content> !
               </ng-container>`
})
export class MultipleHelloComponent implements OnInit {
    @Input()
    times: number;

    items: any[] = [];

    ngOnInit(): void {
        this.items = Array(this.times).fill('');
    }
}
```
Le template du composant itère maintenant pour afficher plusieurs fois le message en utilisant la transclusion. L'utilisation du composant se fait de la manière suivante :

```html
<multiple-hello [times]='5'>Sébastien</multiple-hello>
```

L'utilisation précédente du composant demande d'afficher 5 fois le message et de transclure le label "Sébastien". Le résultat est le suivant :

```html
Multiple Hello ! Multiple Hello ! Multiple Hello ! Multiple Hello ! Multiple Hello Sébastien !
```
La transclusion n'a été effectuée qu'une seule fois. En effet, la balise `ng-content` ne permet d'effectuer une transclusion qu'une seule fois, même si elle est utilisée au sein d'un ngFor.

## ng-template

Pour palier à ce genre de besoin, Angular introduit la notion de template, via la directive <a title="Ng Template Guide" href="https://angular.io/guide/structural-directives#the-ng-template" target="_blank">ng-template</a>. Cette notion est utilisée dans plusieurs directives structurelles (ngIf, ngFor, ngSwitch) et permet de définir un template, qui ne sera pas affiché mais qui pourra être utilisé plus tard ou à d'autres endroits.

Prenons maintenant un exemple de composant un peu plus intéressant afin d'illustrer cette notion : le  `MultiSelectComponent`. L'idée de ce composant est de rendre une liste d'éléments sélectionnable. Voici le code :

```typescript
export class MultiSelectComponent {
    @Input() options: ISelectable<any>[]

    @Output() onSelectionChangeEvent: EventEmitter<ISelectable<any>[]> = new EventEmitter();

    onOptionClicked(option: ISelectableModel<any>) {
        option.selected = !option.selected;

        this.onSelectionChangeEvent.emit(this.options);
    }
}
```
 Le composant prend en entrée une liste d'éléments et utilise un event emitter pour renvoyer les éléments à chaque modification de sélection. Le template du composant est le suivant :

```html
<ul *ngFor='let option of options'>
    <li>
        <input type='checkbox' [value]='option.selected' [checked]='option.selected' (click)='onOptionClicked(option)' />
        <ng-content'></ng-content>
    </li>
</ul>
```
Comme vu précédemment, cette vue ne peut pas fonctionner. Le problème vient ici de l'utilisation de la transclusion qui ne sera appliquée que sur un seul élément.

### Déclaration du conteneur

Il va donc falloir modifier ce composant pour accepter un template en entrée et l'injecter à la bonne position.

La référence vers le template fourni par le composant parent se fait simplement en déclarant une propriété de type <a title="TemplateRef documentation" href="https://angular.io/api/core/TemplateRef" target="_blank">TemplateRef</a>, décoré par <a title="Content child documentation" href="https://angular.io/api/core/ContentChild" target="_blank">@ContentChild</a> :

```typescript
export class MultiSelectComponent {
  @ContentChild(TemplateRef) template: TemplateRef<any>;

   [...]
}
```

L'utilisation du template se fait alors via `ng-container`. On doit donc remplacer `ng-content` par `ng-container`, en fournissant le template à appliquer via l'input `ngTemplateOutlet` :

```html
<ul *ngFor='let option of options'>
    <li>
      <input type='checkbox' [value]='option.selected' [checked]='option.selected' (click)='onOptionClicked(option)' />
      <ng-container [ngTemplateOutlet]='template' [ngTemplateOutletContext]='{ $implicit: option }'></ng-container>
    </li>
</ul>
```

 Il est intéressant de noter que l'on spécifie un contexte à passer au template, via l'input `ngTemplateOutletContext`. De cette manière, le template fourni par le parent pourra faire référence au contexte.

### Utilisation du composant

L'utilisation du composant change puisqu'on doit maintenant lui passer un template :

```html
<multi-select [options]='items' (onSelectionChanged)='onItemsSelectionChanged($event)'>
    <ng-template let-option>
        {{option.title}}
    </ng-template>
</multi-select>
```

Deux points sont importants ici. Le premier est la déclaration d'un template via `ng-template`. C'est ce template qui sera ensuite utilisé par le composant et qui sera injecté à l'emplacement du `ng-container`. Le deuxième point important est l'attribut `let-option`. La syntaxe `let-*` permet de créer une référence vers le contexte du template, déclaré via le `ngTemplateOutletContext`. De cette manière, on pourra accéder depuis le template à une variable nommée `option` qui correspondra au contexte passé par le composant MultiSelect.

Et voilà ! De cette manière nous avons pu utiliser un mécanisme similaire à la transclusion dans un ngFor.

Il est intéressant de noter que vous avez déjà utilisé ce mécanisme sans le savoir, via `*ngFor`. En fait, la syntaxe `*ngFor` est simplement un sucre syntaxique vers l'utilisation de `ngFor` sur un template. Les deux syntaxes suivantes fonctionnent donc exactement de la même manière :

```html
<ul *ngFor='let option of options'>
    <li>{{option}}</li>
</ul>

<ul>
    <ng-template ngFor let-option [ngForOf]="options">
        <li>{{option}}</li>
    </ng-template>
</ul>
```

Bonnes transclusions !
