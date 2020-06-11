---
layout: post
title: 'A la découverte des décorateurs TypeScript'
date: 2018-02-27
categories: javascript
resume: 'Un décorateur TypeScript est une annotation permettant d''altérer du code au moment de sa déclaration. Voyons ensemble comment ils fonctionnent et comment en créer.'
tags: typescript decorators
---
Un décorateur TypeScript est une annotation permettant d'altérer du code au moment de sa déclaration. On parle ici d'une syntaxe de méta-programmation (meta-programming syntax) puisqu'il est possible d'ajouter du comportement sans code.

Puisqu'un exemple est toujours plus parlant qu'une explication, comparons la déclaration d'un composant en AngularJS et en Angular. La version AngularJS nécessite le code suivant :

```javascript
class MoreLessComponentController {
    […]
}

class MoreLessComponent implements ng.IComponentOptions {
    controllerAs = 'vm';
    controller = MoreLessComponentController;
    bindings = {
        value: '='
    };
 
    template = '[…]';
}
 
angular.module('myapp').component('moreLess', new MoreLessComponent());
```

Ce code contient trois portions : le controller _MoreLessComponentController_ contenant la logique du composant, le composant _MoreLessComponent_ contenant la description du composant (son template, ses bindings, etc.) et la déclaration du composant en tant que composant d'un module.

_Si vous êtes intéressé par une explication plus détaillée du code précédent, vous pouvez vous référer à l'article suivant : <a href="https://sebastienollivier.fr/blog/javascript/creer-un-component-angularjs-en-typescript">https://sebastienollivier.fr/blog/javascript/creer-un-component-angularjs-en-typescript</a>_

Le code équivalent en Angular ressemblerait à :

```javascript
@Component({
    selector: 'more-less',
    template: '[…]'
})
class MoreLessComponent {
    @Input() value: string;

    […]
}
```
Ce code déclare une classe contenant la logique du composant. Cette classe est décorée par le décorateur _Component_ permettant de décrire le composant et d'indiquer à Angular que cette classe doit être considérée comme un composant (l'exemple ne prend pas en compte l'enregistrement dans le module qui se fait via le décorateur _NgModule_).

On voit dans cet exemple que l'utilisation du décorateur allège considérablement le code. Il permet d'éviter d'avoir à écrire du code purement technique (déclaration du composant dans notre cas) et de se concentrer sur le code fonctionnel.

Les décorateurs sont notamment beaucoup utilisés par le framework Angular afin de décrire le code : @Component, @Input, @Injectable, @ViewChild, etc.

_Les décorateurs TypeScript sont au stade d'experimental feature. Cela implique qu'il est nécessaire d'activer l'option experimentalDecorators via la commande tsc --experimentalDecorators ou via le fichier tsconfig.json :_

```javascript
{
    "compilerOptions": {
        "experimentalDecorators": true
    }
}
```

_Il existe une proposition de spécification ES7 pour les décorateurs afin qu'ils soient inclus de base dans le langage JavaScript, sans nécessiter de transpileur. Vous pouvez vous référer à ce lien pour plus d'informations : <a href="https://github.com/tc39/proposal-decorators" target="_blank">https://github.com/tc39/proposal-decorators</a>._

## Syntaxe d'un décorateur

Un décorateur est utilisable sur quatre types d'éléments différents : sur une classe, sur une propriété, sur une méthode ou sur un paramètre. Il est également possible d'en appliquer plusieurs sur un même élement.

Un décorateur se déclare de la façon suivante :

```javascript
function monPremierDecorateur(monPremierParametre: string) {
    return ([…]) => {
    } 
};
```

Ce template de code permet de créer un décorateur nommé _monPremierDecorateur_ qui contiendra un paramètre _monPremierParametre_. Ce qui est important de noter est que l'on déclare une factory chargée de renvoyer l'implémentation du décorateur (on peut se passer de la factory mais on ne pourra pas utiliser des paramètres dans ce cas). L'implémentation du décorateur est dépendante du type d'élément ciblé comme on le verra juste après.

L'utilisation de ce décorateur se fait alors simplement en le préfixant par @ :

```javascript
@monPremierDecorateur("c'est super !")
```

Voyons maintenant chaque type de décorateur.

## Décorateur de classe

L'implémentation d'un décorateur de classe prend en paramètre le type de la classe (typé en _Function_ en TypeScript). On peut alors modifier cette classe en y ajoutant par exemple des méthodes :

```javascript
export function debugMeAtRuntime() {
    return (target: Function) => {
        target.prototype.showMe = function() {
            console.debug(JSON.stringify(this));
        }
    } 
};
```

Le décorateur de classe _debugMeAtRuntime_ précédent ajoute automatiquement une méthode _showMe_ (chargée d'afficher dans la console la valeur de la classe) sur le prototype de la classe. Son utilisation se fait alors de la manière suivante :

```javascript
@debugMeAtRuntime()
export class Speaker {
    [...]
}
```

Il est alors possible d'appeler la méthode _showMe_ d'une instance de _Speaker_ (en l'ayant casté en _any_ sinon TypeScript détectera une erreur) :

```javascript
const speaker = new Speaker();
(<any>speaker).showMe();
```

Comme ce décorateur ne prend pas de paramètres, on peut le déclarer sans factory. Cela donne :

```javascript
export function debugMeAtRuntime(target: Function) {
    target.prototype.showMe = function () {
        console.debug(JSON.stringify(this));
    }
};
```

L'utilisation se fait alors sans préciser les parenthèses :

```javascript
@debugMeAtRuntime
export class Speaker {
    [...]
}
```

## Décorateur de propriété

L'implémentation d'un décorateur de propriété prend en paramètres le prototype de la classe ainsi que le nom de la propriété ciblée. Cela permet de pouvoir modifier la propriété via le prototype de la classe :

```javascript
export function envValue(propertyName: string = null) {
    return (target: Object, propertyKey: string) => {
        const setter = function (val) {
            throw 'cannot change value of property flagged with envValue decorator';
        };
        const getter = function () {
            return env[propertyName || propertyKey];
        };
        
        Object.defineProperty(target, propertyKey, {
            get: getter,
            set: setter
        });
    } 
};
```

Le décorateur de propriété _envValue_ précédent permet d'affecter automatiquement une valeur d'environnement (définit dans un objet _env_ déclaré en global) à la propriété.

On définit ici un setter qui effectue simplement un _throw_ (puisque la propriété est considérée en readonly) ainsi qu'un getter qui va renvoyer la valeur d'environnement (en fonction soit du paramètre du décorateur soit du nom de la propriété). L'appel à _Object.defineProperty_ permet alors de surcharger la propriété ciblée en fournissant les nouveaux accesseurs.

Son utilisation se fait alors de la manière suivante :

```javascript
@envValue()
private defaultLanguage: string;
```

Ici, la valeur de la propriété sera égale à la valeur de _env.defaultLanguage_.

```javascript
@envValue("lang")
private defaultLanguage: string;
```

Ici, la valeur de la propriété sera égale à la valeur de _env.lang_.

## Décorateur de méthode

L'implémentation d'un décorateur de méthode prend en paramètres le prototype de la classe, le nom de la méthode cible ainsi qu'un objet de type 
_TypedPropertyDescriptor<T>_ représentant la description de la méthode (_T_ correspondant à la signature de la méthode cible).

Cet objet possède notamment une propriété _value_ pointant vers l'implémentation de la méthode. On peut ainsi capturer ce pointeur puis modifier l'implémentation.

```javascript
export function checkNullOrUndefinedParams(throwIfDetected: boolean = false) {
    return (target: Object, methodKey: string, descriptor: TypedPropertyDescriptor<any>) => {
        const originalMethod = descriptor.value;
        
        descriptor.value = function (...args: any[]) {
            args.forEach(arg => {
                if (!arg) {
                    if (throwIfDetected) {
                        throw 'the parameter is null or undefined';
                    } else {
                        console.warn('the parameter is null or undefined');
                    }
                }
            });
            return originalMethod.apply(this, args);
        };
    }
};
```

Le décorateur de méthode _checkNullOrUndefinedParams_ précédent permet de vérifier que tous les paramètres sont renseignés et, si ce n'est pas le cas, d'afficher un warning dans la console ou de lancer une exception en fonction de la configuration du décorateur.

Le code _const originalMethod = descriptor.value;_ permet de garder une référence vers l'implémentation de la méthode. Le code _descriptor.value = function (...args: any[]) {_ permet alors de redéfinir la méthode en vérifiant les paramètres puis en appelant la méthode d'origine, via _return originalMethod.apply(this, args);_

L'utilisation du décorateur se fait via la syntaxe suivante :

```javascript
@checkNullOrUndefinedParams()
sayHello(language: string) {
    […]
}
```

Deux choses sont importantes à noter lorsque l'on crée des décorateurs de méthode. Le premier point est de ne pas utiliser d'arrow function (_=>_) lors de la surcharge de _descriptor.value_. Si vous le faites, la valeur du _this_ correspondra alors au décorateur alors que l'on souhaite qu'elle corresponde à l'instance de la classe. Le deuxième point est qu'il est important de ne pas recréer un _descriptor_, mais de modifier l'existant, de manière à ce que l'on puisse appliquer plusieurs décorateurs sur une méthode et que chaque décorateur effectue la modification qu'il souhaite.

## Décorateur de paramètre

L'implémentation d'un décorateur de paramètre prend en paramètres le prototype de la classe, le nom de la méthode cible ainsi que l'index de la propriété décorée. Ce type de décorateur ne permet pas de modifier la valeur d'un paramètre (puisqu'il est exécuté en amont de l'appel à la méthode) mais doit être utilisé afin d'ajouter des metadonnées qui seront utilisées par un décorateur de méthode.

```javascript
export function mapToEnvIfEmpty(envKey: string) {
    return (target: Object, propertyKey: string, propertyIndex: number) => {
        const propertyMapName = `__${propertyKey}_params_envMapping`;
        
        if (!target[propertyMapName]) {
            target[propertyMapName] = [];
        }
        
        target[propertyMapName].push({ index: propertyIndex, envKey: envKey })
    }
}
```

Le décorateur de paramètre _mapToEnvIfEmpty_ précédent permet de remplacer la valeur d'un paramètre, si elle n'est pas définie, par un valeur d'environnement. Pour cela, il va ajouter une propriété sur le prototype de la classe afin de garder la mapping entre l'index de la propriété à remplacer et la clef de l'environnement.

En l'état, ce décorateur n'est pas suffisant. Il faut le combiner à un décorateur de méthode, dont le rôle va être de surcharger l'implémentation de la méthode pour modifier les valeurs des paramètres en fonction des metadonnées déclarées par les décorateurs de paramètre.

```javascript
export function mapParametersToEnv() {
    return (target: Object, propertyKey: string, descriptor: TypedPropertyDescriptor<any>) => {
        const originalMethod = descriptor.value;
        const propertyMapName = `__${propertyKey}_params_envMapping`;
        descriptor.value = function (...args: any[]) {
            if (target[propertyMapName]) {
                target[propertyMapName].forEach(prop => {
                    if (!args[prop.index]) {
                        args[prop.index] = env[prop.envKey];
                    }
                });
            }
        
            return originalMethod.apply(this, args);
        };
    }
}
```

L'utilisation de ces décorateurs se fait alors via la syntaxe suivante :

```javascript
@mapParametersToEnv()
sayAnotherThing(sentence: string, @mapToEnvIfEmpty("lang") language: string)
```

Voilà ! Comme nous l'avons vu, les décorateurs permettent d'encapsuler de façon extrêmement élégante du code technique, voir fonctionnel. N'hésitez pas à créer les votres, votre code n'en sera que plus lisible.

Bon décorateurs !
