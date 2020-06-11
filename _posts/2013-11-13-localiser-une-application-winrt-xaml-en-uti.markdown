---
layout: post
title: 'Localiser une application WinRT XAML en utilisant le binding aux ressources'
date: 2013-11-13
categories: winrt
resume: '.NET propose un mécanisme de localisation via un système de ressources (fichiers resx ou resw). Découvrons comment en profiter dans une application WinRT.'
tags: localize xaml l10n
---
**[Update : Une nouvelle solution plus élégante (et avec autocompletion et vérification à la compilation) a été mise en place. Pour plus d'informations : <a href="https://sebastienollivier.fr/blog/winrt/localiser-une-application-winrt-xaml-en-uti-2" target="_blank">https://sebastienollivier.fr/blog/winrt/localiser-une-application-winrt-xaml-en-uti-2</a>]**

Avec la mise en place des stores (Windows Phone, Windows 8, iOS, Android, etc.) et la possibilité de publier une application dans le monde entier, la localisation des applications est devenue fondamentale.

Pour localiser une application Windows Phone, on peut binder directement des ressources aux propriétés des contrôles des vues. Cela donne un contrôle déclaré comme ci-après, pour afficher la ressource localisée `ApplicationTitle` du fichier de ressources `CommonResources` :

```xml
<TextBlock Text="{Binding CommonResources.ApplicationTitle, 
                       Source={StaticResource LocalizedStrings}}" />
```

_Pour plus d'informations sur la manière de localiser une application Windows Phone : <a href="http://blogs.msdn.com/b/pierreca/archive/2010/09/21/internationalisez-votre-application-windows-phone-7.aspx" target="_blank" title="Internationalisez votre application Windows Phone 7">http://blogs.msdn.com/b/pierreca/archive/2010/09/21/internationalisez-votre-application-windows-phone-7.aspx</a>_

Malheureusement avec WinRT, on ne peut pas binder des ressources aux contrôles des vues. A la place, on peut soit utiliser la technique des Uid, soit utiliser la classe `ResourceLoader` pour récupérer la ressource localisée et l'appliquer manuellement.

_Pour plus d'informations sur la manière de localiser une application WinRT de manière "classique" : <a href="http://win8dev.fr/comment-localizer-votre-application/" target="_blank" title="Comment localizer votre application Metro XAML ?">http://win8dev.fr/comment-localizer-votre-application/</a>_

Bien que cela fonctionne, les deux techniques me semblent moins efficaces que ce qu'on a à disposition sur Windows Phone. Passer par les Uid nécessite de définir un Uid à chaque contrôle, crée un couplage fort entre l'Uid du contrôle et la clef de la ressource (si on change l'Uid du contrôle, la clef de ressource doit changer et inversement) et ne permet pas de partager les fichiers de ressources entre plateformes (par exemple, même fichier de ressources pour les versions Windows Phone et Windows 8 d'une application). La technique du `ResourceLoader` oblige à écrire du code-behind, ce qu'on cherche généralement à éviter en utilisant le pattern MVVM.

J'ai donc pris le temps de mettre en place une solution alternative, permettant d'avoir un mécanisme plus ou moins semblable à celui de Windows Phone, c'est à dire faire du binding de ressources directement dans les vues. Le principe est ici de passer par un `Converter` qui, pour un fichier de ressource et une clef, va renvoyer la ressource localisée.

La première étape consiste à créer un helper dont l'objectif va être de récupérer les ressources localisées :

```csharp
public static class ResourceHelper
{
    private static Dictionary<string, ResourceLoader> resources;

    public static string GetLocalizedString(string resourceFile, string resourceKey)
    {
        if (resources == null)
        {
            resources = new Dictionary<string, ResourceLoader>
            {
                 { "CommonResources", new ResourceLoader("CommonResources") },
                 { "DownloadTaskDetailPageResources", new ResourceLoader("DownloadTaskDetailPageResources") },
                 { "DownloadTasksPageResources", new ResourceLoader("DownloadTasksPageResources") },
                 { "LoginPageResources", new ResourceLoader("LoginPageResources") },
                 { "AboutPageResources", new ResourceLoader("AboutPageResources") },
                 { "AboutSearchEnginePageResources", new ResourceLoader("AboutSearchEnginePageResources") },
                 { "TorrentSearcherPageResources", new ResourceLoader("TorrentSearcherPageResources") }
            };
        }

        if (resources.ContainsKey(resourceFile))
        {
            return resources[resourceFile].GetString(resourceKey);
        }

        return string.Empty;
    }
}
```

Le principe est simple. Le helper possède un dictionnaire de `ResourceLoader` (pour avoir une instance par fichier de ressources) et une méthode `GetLocalizedString` qui, pour une clef de ressource et un nom de fichier de ressources, va retourner la ressource localisée.

On va ensuite créer un converter qui n'aura qu'à appeler le helper précédemment créé :

```csharp
public class ResourceConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, string language)
    {
       return ResourceHelper.GetLocalizedString(value.ToString(), parameter.ToString());
    }

    public object ConvertBack(object value, Type targetType, object parameter, string language)
    {
        throw new NotImplementedException();
    }
}
```

Le paramètre `value`, sur lequel le binding se fait, correspond au nom du fichier de ressources. Le paramètre `parameter`, correspondant au `ConverterParameter`, permet de renseigner la clef de ressource.

On doit maintenant exposer le nom des fichiers de ressources pour pouvoir les binder via le converter :

```csharp
public class LocalizedStrings
{
    public string CommonResources { get { return "CommonResources"; } }
    public string DownloadTaskDetailPageResources { get { return "DownloadTaskDetailPageResources"; } }
    public string DownloadTasksPageResources { get { return "DownloadTasksPageResources"; } }
    public string LoginPageResources { get { return "LoginPageResources"; } }
    public string AboutPageResources { get { return "AboutPageResources"; } }
    public string AboutSearchEnginePageResources { get { return "AboutSearchEnginePageResources"; } }
    public string TorrentSearcherPageResources { get { return "TorrentSearcherPageResources"; } }
}
```

Au niveau de la vue, on se retrouve alors avec la syntaxe suivante :

```xml
<converters:ResourceConverter x:Key="ResourceConverter"/>
<w8:LocalizedStrings x:Key="LocalizedStrings" />

[...]

<TextBlock Text="{Binding CommonResources, Source={StaticResource LocalizedStrings}, 
    Converter={StaticResource ResourceConverter}, ConverterParameter=ApplicationTitle}" />
```

Le binding se fait sur une propriété de la classe `LocalizedStrings` pour indiquer que l'on souhaite utiliser le fichier de ressources `CommonResources_` et le _ConverterParameter_ permet de spécifier que l'on souhaite récupérer la ressource `ApplicationTitle`.

Avantages de cette méthode :

  * Je n'ai pas à modifier mes clefs de ressources (si mes fichiers de ressources existent déjà)
  * Toute la localisation se fait au niveau de la vue
  * La syntaxe du binding est quasiment identique à la syntaxe Windows Phone
  * L'éditeur de vue Visual Studio et Blend interprètent le converter et affichent la ressource localisée

Désavantage de cette méthode :

  * La clef de ressource est spécifiée sous forme de chaîne de caractères, ce qui peut provoquer des erreurs de frappe

Bonne localisation !

**[Update : Une nouvelle solution plus élégante (et avec autocompletion et vérification à la compilation) a été mise en place. Pour plus d'informations : <a href="https://sebastienollivier.fr/blog/winrt/localiser-une-application-winrt-xaml-en-uti-2" target="_blank">https://sebastienollivier.fr/blog/winrt/localiser-une-application-winrt-xaml-en-uti-2</a>]**


