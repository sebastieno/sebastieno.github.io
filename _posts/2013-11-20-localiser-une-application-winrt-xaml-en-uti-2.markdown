---
layout: post
title: 'Localiser une application WinRT XAML en utilisant le binding aux ressources... en encore mieux !!!'
date: 2013-11-20
categories: winrt
resume: '.NET propose un mécanisme de localisation via un système de ressources (fichiers resx ou resw). Découvrons comment en profiter dans une application WinRT.'
tags: localize xaml l10n
---
On a vu, dans <a href="http://sebastienollivier.fr/blog/winrt/localiser-une-application-winrt-xaml-en-utilisant-le-binding-aux-ressources/" target="_blank">l'article précédent</a>, comment faire pour binder des ressources directement dans les vues WinRT (et ne pas être obligé de localiser notre application en utilisant le binding via Uid ou passer par du code-behind). Le principe consiste à passer par un `Converter` qui, pour une clef de ressource et un nom de fichier, va récupérer une ressource localisée via la classe `ResourceLoader`.

L'utilisation de ce converter est plutôt appréciable mais atteint vite ses limites lorsque l'on travaille sur une application possédant beaucoup de ressources. Le désavantage de ce converter est qu'il faut renseigner la clef de ressource sous forme d'une chaîne de caractères ce qui implique de n'avoir ni l'autocompletion ni la vérification à la compilation (ce qui est très dommageable en terme de confort d'utilisation et de productivité).

La solution idéale serait de reproduire le fonctionnement classique utilisé pour les fichiers resx (en ASP.NET, Windows Phone, etc.), c'est à dire un `Custom Tool` (PublicResxFileCodeGenerator) chargé de générer une classe par fichier resx qui expose les ressources via la classe _ResourceManager_. Comme le développement et le déploiement d'un `Custom Tool` est assez pénible, j'ai préféré créer un template T4.

Pour chaque fichier de ressource resw du projet, le template va générer une classe ayant la structure suivante :

```csharp
public class CommonResources 
{
    private static Windows.ApplicationModel.Resources.ResourceLoader resourceLoader;
    public static Windows.ApplicationModel.Resources.ResourceLoader ResourceLoader
    {
        get
        {
	        if(resourceLoader == null)
	        {
	            resourceLoader = new Windows.ApplicationModel.Resources.ResourceLoader("CommonResources");
	        }

            return resourceLoader;
        }
    }

    ///<summary>
    ///
    ///</summary>
    public string ApplicationTitle 
    { 
        get { return ResourceLoader.GetString("ApplicationTitle"); }
    }
}
```

_Une version Windows 8.1 du template existe, la seule différence se situant à la ligne 10 (utilisation de ResourceLoader.GetForCurrentView(...) au lieu de new ResourceLoader(...))._

La classe contient un singleton représentant une instance de `ResourceLoader` et expose chaque clef de ressource sous la forme d'une propriété qui, via le `ResourceLoader`, va retourner la ressource localisée.

Pour pouvoir utiliser ces classes de ressources dans nos vues, il faut alors les exposer de la manière suivante :

```csharp
public class LocalizedStrings
{
    public CommonResources CommonResources { get { return new CommonResources(); } }
}
```

Au niveau de la vue, on se retrouve avec la syntaxe suivante :

```xml
w8:LocalizedStrings x:Key="LocalizedStrings" />

[...]

<TextBlock Text="{Binding CommonResources.ApplicationTitle, Source={StaticResource LocalizedStrings}}" />
```

Le binding se fait de manière classique (c'est à dire comme pour les fichiers resx, sur ASP.NET, Windows Phone, etc.). On peut profiter de l'autocompletion et de la vérification à la compilation :)

Ce template est disponible sur GitHub à l'adresse suivante : <a href="https://github.com/sebastieno/PublicResWFileCodeGenerator" target="_blank">https://github.com/sebastieno/PublicResWFileCodeGenerator</a>

Pour l'utiliser, il suffit d'inclure le template dans le projet contenant les fichiers resw, modifier la valeur de la variable `resourcesNamespace` pour définir le namespace des classes générées et c'est tout !

Le template est aussi disponible sous la forme d'un package nuget : <a href="https://www.nuget.org/packages/PublicResWFileCodeGenerator.W8" target="_blank">PublicResWFileCodeGenerator For Windows 8</span></a> <a href="https://www.nuget.org/packages/PublicResWFileCodeGenerator.W8.1" target="_blank"> PublicResWFileCodeGenerator For Windows 8.1</span></a>

Très bonne localisation ! 


