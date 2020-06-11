---
layout: post
title: 'Expression Tree'
date: 2011-11-20
categories: net
resume: 'Les arbres d''expressions .NET sont méconnus mais sont pourtant utilisés régulièrement (notamment par EF). Voyons ensemble ce qu''ils apportent.'
tags: net expression expression-tree
---
Bien que régulièrement utilisés par la plupart d'entre nous (en tout cas pour ceux qui ont la chance d'utiliser EF), les arbres d'expressions sont pourtant méconnus. On va donc voir dans ce post ce que sont les arbres d'expressions et dans quel contexte ils sont utilisés.

Il est nécessaire de comprendre le fonctionnement des expressions lambda. Pour cela, n'hésitez pas à vous référer au post précédent: <a href="http://sebastienollivier.fr/blog/net/delegate-lambda-linq/" rel="bookmark" target="_blank">Delegate, Lambda & Linq</a>.

## Expression&lt;Func&lt;T&gt;&gt;

En plus de l'apport de la classe `Func<>`, le Framework .NET 3.5 introduit une notion nouvelle dans le Framework .NET : les arbres d'expressions, symbolisés par la classe `Expression<Func<>>`.

Un arbre d'expressions est la représentation sous forme d'un arbre un code exécutable.

![Expression Tree Viewer](https://farm9.staticflickr.com/8481/8211254146_c9a26049a6_o.png =363x521 "Expression Tree Viewer")

_Expression Tree Viewer_ est un projet disponible avec Visual Studio 2008. Il permet de représenter un arbre d'expression dans un contrôle de type `Treeview`.

Contrairement à `Func<>`, un arbre d'expressions n'est pas transformé en délégué à la compilation. Il est nécessaire de demander explicitement la compilation d'un arbre d'expressions pour qu'un délégué soit généré.

```csharp
static void Main(string[] args)
{
    Expression<Func<DateTime, string>> speakerTree =
        date => string.Format("Hello, I am an expression tree, at {0} !",
                               date.ToShortTimeString());

    Func<DateTime, string> speaker = speakerTree.Compile();

    Console.WriteLine(speaker(DateTime.Now));

    Console.ReadLine();
}
```

Le code précédent illustre l'utilisation d'un arbre d'expression.

Un arbre d'expressions est créé via une expression lambda. La fonction représentée par l'arbre d'expressions prend en paramètre une date et renvoie un string.

Il est intéressant de noter qu'aucun délégué n'est généré à la compilation. L'appel à la méthode `Compile` va déclencher la création d'un délégué à l'exécution. Pour cela, de l'injection de MSIL va être effectué.

Le résultat du code précédant est le suivant :

![Output](https://farm9.staticflickr.com/8347/8211254182_9800416f45_o.png =380x65 "Output")

## Création d'un Expression Tree via l'API

Nous avons vu comment créer un arbre d'expression en utilisant une expression lambda. Il est possible de créer directement un arbre d'expression en utilisant l'API mis à disposition par le Framework .NET 3.5.

Pour utiliser l'API, il est nécessaire d'ajouter le `using` suivant:

```csharp
using System.Linq.Expressions;
```

Le but du code qui va suivre est de représenter l'expression lambda utilisée précédemment sous la forme d'un arbre d'expression.

Pour rappel, l'expression lambda utilisée est la suivante:

```csharp
date => string.Format("Hello, I am an expression tree, at {0} !",
                  date.ToShortTimeString());
```

La première tâche va consister à représenter le paramètre `date`. Pour cela, nous allons faire appel à la méthode `Expression.Parameter` en indiquant le type du paramètre ainsi que son nom.

```csharp
// Déclaration d'un paramètre de type DateTime nommé "date"
ParameterExpression date = Expression.Parameter(typeof(DateTime), "date");
```

Nous allons maintenant utiliser ce paramètre pour faire appel à la méthode `ToShortTimeString` utilisé par `string.Format`. Pour cela, il faut récupérer, par réflexion, la méthode `ToShortTimeString` et ensuite faire appel à la méthode `Expression.Call`. Cette méthode attend en paramètres une expression symbolisant l'élément sur lequel sera effectué l'appel à la méthode et la méthode à appeler.

```csharp
// Récupération de la méthode ToShortTimeString par reflection
MethodInfo toShortTimeStringMethod = typeof(DateTime).GetMethod("ToShortTimeString");
// Récupération de la méthode ToShortTimeString par reflection
MethodCallExpression toShortTimeStringMethodCall =
                               Expression.Call(date, toShortTimeStringMethod);
```

Pour déclarer une constante contenant le string à afficher, nous allons appeler à la méthode `Expression.Constant`.

```csharp
// Déclaration d'une constante
ConstantExpression format = Expression.Constant(
                "Hello, I am an expression tree created from scratch, at {0} !");
```

Il reste maintenant à déclarer l'appel à la méthode `string.Format`. Nous allons récupérer par réflexion la méthode `Format`. Un appel sera ensuite fait à la méthode `Expression.Call`.

```csharp
// Récupération de la méthode Format par reflection
MethodInfo formatMethod = typeof(string).GetMethod("Format",
                     new[] { typeof(string), typeof(DateTime) });
// Déclaration d'un appel à la méthode Format avec paramètres
MethodCallExpression body = Expression.Call(formatMethod, format,
                                    toShortTimeStringMethodCall);
```

Il faut maintenant assembler les différents éléments. La méthode `Expression.Lambda` crée un arbre d'expressions à partir d'expressions représentant les paramètres et d'expressions représentant les instructions à exécuter.

```csharp
// Déclaration de l'arbre d'expressions
Expression<Func<DateTime, string>> speakerTree =
                    Expression.Lambda<Func<DateTime, string>>(body, date);
```

Ci-dessous le code complet:

```csharp
static void Main(string[] args)
{
    // Déclaration d'un paramètre de type DateTime nommé "date"
    ParameterExpression date = Expression.Parameter(typeof(DateTime), "date");

    // Récupération de la méthode ToShortTimeString par reflection
    MethodInfo toShortTimeStringMethod =
                    typeof(DateTime).GetMethod("ToShortTimeString");
    // Déclaration d'un appel à la méthode ToShortTimeString sur le paramètre date
    MethodCallExpression toShortTimeStringMethodCall = Expression.Call(date,
                toShortTimeStringMethod);

    // Déclaration d'une constante
    ConstantExpression format = Expression.Constant(
                "Hello, I am an expression tree created from scratch, at {0} !");

    // Récupération de la méthode Format par reflection
    MethodInfo formatMethod = typeof(string).GetMethod("Format",
                 new[] { typeof(string), typeof(DateTime) });
    // Déclaration d'un appel à la méthode Format avec paramètres
    MethodCallExpression body = Expression.Call(formatMethod, format,
                 toShortTimeStringMethodCall);

    // Déclaration de l'arbre d'expressions
    Expression<Func<DateTime, string>> speakerTree =
                 Expression.Lambda<Func<DateTime, string>>(body, date);

    Func<DateTime, string> speaker = speakerTree.Compile();

    Console.WriteLine(speaker(DateTime.Now));
    Console.ReadLine();

}
```

Pour information, la méthode `ToString` de `Expression<Func<>>` permet d'afficher l'expression représenté par l'arbre d'expressions.

Contrairement aux délégués, l'utilisation d'arbres d'expressions n'est pas _type-safe_ puisque la génération du délégué se fait à l'exécution.

En modifiant le type du paramètre `date`, le code continue de compiler alors qu'une exception est lancé à l'exécution.

![ArgumentException](https://farm9.staticflickr.com/8342/8210164771_d92f736dc4_o.png =624x160 "ArgumentException")

## Linq to Entities

_Linq to Entities_ permet de requêter des données en utilisant les expressions lambda.

Le code ci-dessous illustre l'utilisation de la syntaxe _Linq to Entities_. Il permet de récupérer tous les produits dont la date de vente démarre après le 1 janvier 2006 (_utilisation de la base AdventureWorks_).

```csharp
static void Main(string[] args)
{
    using (AWEntities entities = new AWEntities())
    {
        foreach (Product product in entities.Products.Where(
                     p => p.SellStartDate > new DateTime(2006, 1, 1)))
        {
            Console.WriteLine(product.Name);
        }
    }
}
```

La syntaxe est identique à la syntaxe _Linq to Object_. La seule différence se trouve au niveau de l'exécution du requêtage.

_Linq to Object_ requête des données en mémoire, l'exécution d'une requête s'effectue donc immédiatement.

_Linq to Entities_ requête des données en base, la problématique est donc plus compliquée. Pour des raisons de performances, _Linq to Entities_ utilise l'exécution différée, c'est-à-dire que l'exécution ne s'effectue que lorsque l'on itère sur la collection.

```csharp
static void Main(string[] args)
{
    using (AWEntities entities = new AWEntities())
    {
        var query = entities.Products.Where(p => p.SellStartDate >
                                    new DateTime(2006, 1, 1));
        query = query.OrderBy(p => p.ProductNumber);

        foreach (Product product in query)
        {
            Console.WriteLine(product.Name);
        }
    }

    Console.ReadKey();
}
```

Le code ci-dessus illustre l'exécution différée. L'exécution de la requête n'est ni effectuée à l'appel du `Where` ni à l'appel du `OrderBy`. Elle se fait au niveau du `foreach`, lorsque l'on démarre l'itération. Cela permet de n'effectuer qu'une seule requête SQL, contenant à la fois le `Where` et le `OrderBy`.

Si on comparait le même code en _Linq to Object_, la méthode `Where` aurait déclenché une requête de filtre puis la méthode `OrderBy` aurait déclenché une requête de tri. Le `foreach` n'aurait déclenché aucun requêtage.

Pour pouvoir utiliser ce mécanisme d'exécution différée, _Linq to Entities_ ne travaille pas directement sur l'interface `IEnumerable` mais sur l'interface `IQueryable`, ci-dessous la définition de cette interface:

```csharp
public interface IQueryable : IEnumerable
{
    Type ElementType { get; }
    Expression Expression { get; }
    IQueryProvider Provider { get; }
}
```

L'interface `IQueryable` possède une propriété `Expression` de type `Expression` qui correspond à l'arbre d'expressions représentant la requête à effectuer.

A chaque fois qu'on appelle une méthode d'extension _Linq_, l'arbre d'expressions du `IQueryable` va être enrichi.

En reprenant le code précédent, voici comment _Linq to Entities_ va effectuer la requête:

```csharp
var query = entities.Products.Where(p => p.SellStartDate > new DateTime(2006, 1, 1));
```

L'arbre d'expressions créé à partir de cette instruction est le suivant:

![Expression Tree](https://farm9.staticflickr.com/8344/8211254110_1378d324fb_o.png =479x623 "Expression Tree")

L'encadré 1 correspond à la propriété `p.SellStartDate`. L'encadré 2 défini la création de la date 01/01/2006. L'encadré 3 permet d'effectuer l'opération `GreaterThan` entre la propriété et la date.

Enfin, l'encadré 4 intègre l'opération de l'encadré 3 dans une clause `Where`.

L'instruction suivante décrit une opération de trie.

```csharp
query = query.OrderBy(p => p.ProductNumber);
```

L'arbre d'expressions s'enrichit et devient:

![Expression Tree](https://farm9.staticflickr.com/8057/8211254050_2433d8a330_o.png =436x536 "Expression Tree")

L'encadré 1 correspond à l'opération `Where` déclarée précédemment. L'encadré 2 défini un accès à la propriété `p.ProductNumber`.

Enfin l'encadré 3 décrit une opération `OrderBy` sur la propriété de l'encadré 2.

A ce moment de l'exécution du code, aucune requête SQL n'a été effectuée. Nous avons uniquement créé un arbre d'expressions représentant nos opérations.

L'instruction suivante déclenche l'itération:

```csharp
foreach (Product product in query)
```

L'itération allant débutée, _Linq to Entities_ doit exécuter une requête SQL pour récupérer les données. Contrairement à ce que nous avons vu précédemment, _Linq to Entities_ ne va pas demander la compilation de l'arbre d'expressions pour générer un délégué. Il va transformer l'arbre d'expressions détenu par le `IQueryable` en syntaxe SQL pour générer une requête. Cette requête sera alors exécuter pour que les données puissent être récupérées.

![IQueryable](https://farm9.staticflickr.com/8203/8210164573_b39c5ccca7_o.png =662x298 "IQueryable")

Ci-dessus la requête générée et exécutée correspondant à notre arbre d'expressions.

Nous avons pu voir ce que sont les arbres d'expressions et comment les utiliser. Nous avons aussi constaté que le cœur du système de requêtage de _Linq to Entities_ se base sur cette nouveauté. Bien que les arbres d'expressions soient assez difficiles à appréhender, son utilisation peut se révéler très intéressante dans certains scénarii. Je vous laisse imaginer tout le potentiel !

A vous de jouer ;)

