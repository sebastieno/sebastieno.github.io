---
layout: post
title: 'Delegate, Lambda & Linq'
date: 2011-11-19
categories: net
resume: 'Découvrons l''évolution des délégués à travers les différentes versions de .NET, depuis leur introduction jusqu''à la syntaxe Linq.'
tags: net delegate lambda linq
---
Le Framework .NET 3.5 a apporté beaucoup de nouveautés importantes. Une de ces nouveautés est l'arbre d'expression ou _expression tree_. Je me suis rendu compte que peu de personnes connaissent cette notion, et très peu de personnes la maitrisent.

Avant de voir les arbres d'expressions, il est nécessaire de comprendre les expressions lambda et la syntaxe Linq. A travers ce premier post, je vais présenter l'évolution des délégués à travers les différentes versions de .NET, depuis l'introduction des délégués jusqu'à la syntaxe Linq. <a href="https://sebastienollivier.fr/blog/net/arbre-expression" target="_blank">Nous verrons les arbres d'expressions dans le post suivant</a>.

Le Framework .NET possède le mot clef `delegate`. Un délégué .NET est l'équivalent C++ d'un pointeur à la différence près qu'il est _type-safe_, c’est-à-dire qu'une validation est effectuée à la compilation. Ci-dessous la définition Wikipédia:

<a href="http://en.wikipedia.org/wiki/Delegate_(.NET)" target="_blank"><img  title="Delegate Wikipedia" src="http://farm9.staticflickr.com/8202/8209827301_f72cf7c3cc_o.png" alt="Delegate Wikipedia" width="594" height="115" /></a>

## Méthodes nommées

Avant .NET 2, la seule manière d'utiliser les délégués était de créer des _named methods_ (méthodes nommées).

```csharp
delegate string Speaker(DateTime date);

static string HelloFromFirstMethod(DateTime date)
{
    return string.Format("Hello, I am the first method, at {0} !", date.ToShortTimeString());
}

static string HelloFromSecondMethod(DateTime date)
{

    return string.Format("Hello, I am the second method, the {0} !", date.ToShortDateString());
}

static void Main(string[] args)
{
    Speaker speaker = HelloFromFirstMethod;
    Console.WriteLine(speaker(DateTime.Now));

    speaker = HelloFromSecondMethod;
    
    Console.WriteLine(speaker(DateTime.Now));
    Console.ReadLine();
}
```

Le code précédant illustre l'utilisation des délégués avec des méthodes nommées.

Un délégué nommé `Speaker` est déclaré. Ce délégué prend en paramètre une date et renvoi un string. Deux méthodes statiques, respectant la signature du délégué, retourne un message.

A la création du délégué, on va lui spécifier quelle méthode doit être utilisée. On peut dynamiquement changer la méthode vers laquelle le délégué pointe.

Le résultat du code précédant est le suivant :

![Output](https://farm9.staticflickr.com/8480/8210914988_52bc7d3387_o.png =436x73 "Output")

Si on essaye de faire pointer un délégué vers une méthode ne respectant pas sa signature, le délégué étant _type-safe_, le compilateur va détecter une erreur et la compilation échouera.

![Compilation error](https://farm9.staticflickr.com/8068/8209827355_5f4621a607_o.png =744x282 "Compilation error")

## Méthodes anonymes

Le Framework .NET 2 introduit une nouveauté: les _anonymous methods_ (méthodes anonymes). Il n'est plus nécessaire de créer des méthodes nommées pour utiliser les délégués.

```csharp
delegate string Speaker(DateTime date);

static void Main(string[] args)
{
    Speaker speaker = delegate(DateTime date)
    {
        return string.Format("Hello, I am the first anonymous method, at {0} !", date.ToShortTimeString());
    };

    Console.WriteLine(speaker(DateTime.Now));

    speaker = delegate(DateTime date)
    {
        return string.Format("Hello, I am the second anonymous method, the {0} !", date.ToShortDateString());
    };

    Console.WriteLine(speaker(DateTime.Now));

    Console.ReadLine();
}
```

Le code précédant illustre l'utilisation de méthodes anonymes avec un délégué.

Un délégué nommé `Speaker` est déclaré. Ce délégué prend en paramètre une date et renvoi un string. Il n'y a plus de méthodes statiques déclarées.

A la création du délégué, au lieu de lui spécifier quelle méthode doit être utilisée, on va déclarer une méthode dite anonyme en respectant la syntaxe suivante:

```csharp
delegate()
{
    ...
};
```

Le mot clef `delegate` permet de déclarer cette méthode. Les paramètres doivent correspondre aux paramètres définis par le délégué. Le type retourné par la méthode doit correspondre au type de retour défini par le délégué.

Le résultat du code précédant est le suivant :

![Output](https://farm9.staticflickr.com/8206/8209827265_fffd5f8d91_o.png =459x75 "Output")

De la même manière qu'avec les méthodes nommées, l'utilisation d'un délégué avec des méthodes est _type-safe_.

![Compilation error](https://farm9.staticflickr.com/8207/8210914948_f49d19b34f_o.png =615x129 "Compilation error")

## Expressions lambda

Le Framework .NET 3 introduit une nouvelle syntaxe: les _lambda expressions_. L'idée est de garder la souplesse des méthodes anonymes en simplifiant leur syntaxe.

```csharp
delegate string Speaker(DateTime date);

static void Main(string[] args)
{
    Speaker speaker =
        date => string.Format("Hello, I am the first lambda expression, at {0} !",
                                          date.ToShortTimeString());
    Console.WriteLine(speaker(DateTime.Now));

    speaker =
        date => string.Format("Hello, I am the second lambda expression, the {0} !",
                                          date.ToShortDateString());
    Console.WriteLine(speaker(DateTime.Now));

    Console.ReadLine();
}
```

Le code précédant illustre l'utilisation d'expressions lambda avec un délégué.

La seule différence de code se situe au niveau de la création du délégué. On n'utilise plus le mot clef `delegate` pour déclarer une méthode anonyme. A la place on utilisera la syntaxe des expressions lambda:

```csharp
() => ...
```

L'opérateur `=>` défini l'expression lambda. Les éléments à gauche de l'opérateur correspondent aux paramètres définis par le délégué. Les éléments à droite correspondent aux instructions à exécuter. Le mot clef `return` ne doit plus être utilisé puisqu'un `return` implicite est automatiquement effectué.

Le résultat du code précédant est le suivant :

![Output](https://farm9.staticflickr.com/8203/8209827411_c4e946623f_o.png =475x56 "Output")

De la même manière que vu précédemment, la validité du code est vérifiée à la compilation.

![Compilation error](https://farm9.staticflickr.com/8487/8210914908_123b927ae9_o.png =804x66 "Compilation error")

Il est intéressant de noter qu'à la compilation, l'utilisation de méthodes nommées, de méthodes anonymes ou d'expressions lambda génère le même MSIL.

## Func<>

Le Framework .NET 3.5 apporte une nouvelle encapsulation  des délégués via la classe `Func<>`. Cela permet de créer plus facilement un délégué en allégeant la syntaxe.

```csharp
 
static Func<DateTime, string> speaker;

static void Main(string[] args)
{
    speaker =
        date => string.Format("Hello, I am the first lambda expression, at {0} !",
                                              date.ToShortTimeString());
    Console.WriteLine(speaker(DateTime.Now));

    speaker =
        date => string.Format("Hello, I am the second lambda expression, the {0} !",
                                              date.ToShortDateString());
    Console.WriteLine(speaker(DateTime.Now));

    Console.ReadLine();
}
```

La seule modification apporté au code précédent est la déclaration du délégué. On n'utilise plus le mot clef `delegate` mais `Func<>`. La syntaxe à respecter est la suivante:

```csharp
Func<T1, T2, T3, ..., TResult> speaker;
```

`T1` correspond au type du premier paramètre, `T2` correspond au type du deuxième paramètre, `T3` correspond au type du troisième paramètre, etc. (il peut y avoir jusqu'à 16 paramètres). `TResult` correspond au type de retour du délégué.

Pour déclarer un délégué ne retournant pas de valeur, il faut utiliser `Action` au lieu de `Func<>`.

## Linq

Une nouveauté majeure apportée par le Framework .NET 3.5 est l'ajout de _Linq_ (Language Integrated Query). _Linq_ est une syntaxe objet de requêtage.

_Linq_ ajoute un certain nombre de méthodes d'extensions à l'interface `IEnumerable` (cette interface est implémentée par tous les types énumérables: `List`, `IQueryable`, `Dictionary`, etc.). Ces méthodes d'extensions permettent de faciliter le requêtage et d'alléger le code en utilisant les _lambda expressions_.

Pour utiliser la syntaxe _Linq_, il est nécessaire d'ajouter le `using` suivant :

```csharp
using System.Linq;
```

Linq se décline sous trois formes:

* Linq to Object permettant de requêter des objets au sens C#
* Linq to Entities permettant de requêter des entités en base de données via Entity Framework ou via Linq to SQL
* Linq to Xml permettant de requêter une source XML

Le code suivant illustre l'utilisation de la syntaxe _Linq to Object_ à travers la méthode d'extension `Where`.

```csharp
static void Main(string[] args)
{
    IEnumerable<int> numbers = new List<int> { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9};

    numbers = numbers.Where(i => (i % 3) == 0);

    foreach (int number in numbers)
    {
        Console.WriteLine(number);
    }

    Console.ReadLine();
}
```

Une liste d'entiers est déclarée. On ne souhaite garder que les entiers divisibles par 3. Pour cela, on va utiliser la méthode `Where` permettant de filtrer une liste en fonction d'un délégué.

<a href="http://msdn.microsoft.com/en-us/library/bb534803.aspx" target="_blank">![MSDN Where Linq](https://farm9.staticflickr.com/8199/8209827385_33014de803_o.png =544x15 "MSDN Where Linq")</a>

La méthode `Where` attend en paramètre un délégué. Ce délégué doit avoir un paramètre de type `TSource` (dans notre cas de type `int` puisque notre liste est une liste d'entiers) et renvoyé un booléen. A l'exécution, pour chaque item de la liste, le délégué va être appelé. Si ce délégué renvoie un booléen vrai, l'item de la liste sera ajouté à une nouvelle liste. Après toutes les itérations, cette nouvelle liste sera renvoyée comme type de retour de la méthode `Where`.

Dans notre exemple, le délégué est `i => (i % 3) == 0`. Il renvoi vrai si l'item est divisible par 3.

Ci-dessous le code correspondant sans la syntaxe _Linq to Object_:

```csharp
delegate bool IsInclude(int number);

static void Main(string[] args)
{
    IEnumerable<int> numbers = new List<int> { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9};

    IsInclude isInclude = delegate(int number)
    {
        return (number % 3) == 0;
    };

    List<int> filteredNumbers = new List<int>();

    foreach (int number in numbers)
    {
        if (isInclude(number))
        {
            filteredNumbers.Add(number);
        }
    }

    foreach (int number in filteredNumbers)
    {
        Console.WriteLine(number);
    }

    Console.ReadLine();
}
```

Voici une liste non exhaustive des méthodes d'extensions les plus utilisées:

* `Any` : vérifie si au moins un des items de la liste vérifie une condition

```csharp
bool isDivisibleByThree = numbers.Any(i => (i % 3) == 0);
```
	
* `FirstOrDefault` : retourne le premier item qui vérifie une condition ou la valeur par défaut si aucun item ne vérifie la condition. La méthode `First` retourne le premier item qui vérifie une condition ou lance une exception de type `InvalidOperationException` si aucun item ne vérifie la condition.

```csharp
int firstDivisibleByThree = numbers.FirstOrDefault(i => (i % 3) == 0);
```
	
* `Select` : projette chaque item de la liste dans un nouveau type

```csharp
List<string> stringNumbers = numbers.Select(i => i.ToString());
```

Un aspect intéressant de cette syntaxe est la possibilité de cumuler les ordres de requêtage.

```csharp
numbers.Where(i => (i % 3) == 0).Take(2).Select(i => i.ToString());
```

Le code précédent filtre la liste en ne gardant que les entiers divisibles par 3, ne prend que les deux premiers éléments retournées et les retourne sous forme de string.

Maintenant que nous avons vu les délégués et leur évolution, découvrons <a href="https://sebastienollivier.fr/blog/net/arbre-expression" target="_blank"> les arbres d'expressions</a>.

A bientôt pour la suite...


