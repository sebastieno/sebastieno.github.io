---
layout: post
title: 'Charger le cache de votre API ASPNET Core au démarrage'
date: 2019-03-08
categories: asp-net-mvc
resume: 'Comment peut-on s''assurer que notre cache est hydraté au démarrage de l''API ?'
tags: cache aspnetcore startup
---
Nous avons vu dans les articles précédents <a href="https://sebastienollivier.fr/blog/asp-net-mvc/mettre-en-place-un-cache-local-sur-une-api-aspnetcore">Mettre en place un cache local sur une API ASPNET Core</a> comment mettre en place un cache mémoire local via le nuget <a href="https://www.nuget.org/packages/Microsoft.Extensions.Caching.Memory/" title="Microsoft.Extensions.Caching.Memory Nuget Page" target="_blank">Microsoft.Extensions.Caching.Memory</a> puis <a href="https://sebastienollivier.fr/blog/net/rehydrater-le-cache-local-de-votre-api-aspnetcore-avant-son-expiration">Comment réhydrater le cache avant son expiration</a> (n'hésitez pas à aller jeter un oeil à l'article si vous ne l'avez pas lu).

De cette manière, on peut proposer facilement une API très réactive. Mais il reste une période lors de laquelle notre API ne va pas pouvoir s'appuyer sur le cache et sera donc vulnérable : juste après son démarrage et avant que le cache ne soit hydraté.

Voyons ensemble comment combler supprimer cette période en faisant en sorte que l'API démarre avec un cache prêt.

_Pour illustrer l'importance de ce sujet, je vais vous raconter rapidement une situation qui m'est arrivé. Sur une API utilisant massivement du cache, on a eu un pic d'utilisation non prévue. L'API a commencé à montrer des signes de faiblesses et on a donc décidé d'ajouter une instance. Dès que la nouvelle instance a démarré, elle a commencé à encaisser une partie du traffic. Le problème est qu'elle a été tellement sollicitée avant la mise en place de son cache que son process a planté en boucle, rendant la nouvelle instance totalement inutilisable_.

Le point d'entrée d'une API ASPNET Core est le fichier `Program.cs` :

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateWebHostBuilder(args).Build().Run();
    }

    public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
            .UseKestrel(c => c.AddServerHeader = false)
            .UseStartup<Startup>();
}
```

On y construit une instance de `IWebHostBuilder` qu'on démarre ensuite via l'appel à la méthode `Run`.

L'idée ici est de décomposer cette partie en deux temps : la construction et le démarrage pour y insérer la création du cache.

```csharp
public static void Main(string[] args)
{
    var host = CreateWebHostBuilder(args).Build();
    
    // Création du cache

    host.Run();
}
```

On peut alors insérer le code permettant de créer nos clef de caches :

```csharp
public static void Main(string[] args)
{
    var host = CreateWebHostBuilder(args).Build();
    
    using (var scope = host.Services.CreateScope())
    {
        var memoryCache = scope.ServiceProvider.GetRequiredService<IMemoryCache>();

        this.memoryCache.GetOrCreateAsync("mycachekey", entry => {
            entry.SlidingExpiration = TimeSpan.FromMinutes(30);
            return [...];
        }).Result;
    }
   
    host.Run();
}
```

Ce qui est intéressant de noter dans le code précédent, c'est qu'on a accès à l'injection de dépendances via la propriété `Services` du host. On peut ainsi manipuler directement le cache qui sera utilisé par l'API.

Point également intéressant, il est possible de créer des méthodes `Main` asynchrones depuis C# 7.1. Pour cela, il suffit de spécifier le `LangVersion` de votre projet (dans le fichier *.csproj) :

```xml
<PropertyGroup>
    <LangVersion>latest</LangVersion>
</PropertyGroup>
```

Notre main ressemble maintenant à ça :

```csharp
public static async Task Main(string[] args)
{
    var host = CreateWebHostBuilder(args).Build();
    
    using (var scope = host.Services.CreateScope())
    {
        var memoryCache = scope.ServiceProvider.GetRequiredService<IMemoryCache>();

        await this.memoryCache.GetOrCreateAsync("mycachekey", entry => {
            entry.SlidingExpiration = TimeSpan.FromMinutes(30);
            return [...];
        });
    }
   
    host.Run();
}
```

Le démarrage de l'API sera sensiblement ralentit (puisque des données seront chargées potentiellement depuis une base de données à ce moment) mais elle ne sera accessible (c'est-à-dire qu'elle pourra traiter des requêtes) uniquement lorsque le cache sera prêt. On évite donc le scénario décrit précédemment.

Bon démarrage d'API !

