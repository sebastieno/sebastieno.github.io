---
layout: post
title: 'Mettre en place un cache local sur une API ASPNET Core'
date: 2019-01-16
categories: asp-net-mvc
resume: 'Voyons ensemble comment mettre en place un cache local sur une API ASPNET Core.'
tags: aspnetcore imemorycache configureservices addmemorycache
---
Lorsque l'on commence à traiter le sujet des performances sur une API, on en vient rapidement à l'idée d'utiliser un cache afin d'optimiser l'accès aux données, par exemple en évitant de systématiquement attaquer la base de données.

Un type de cache extrêmement simple à mettre en place est le cache local (en opposition au cache distribué) qui consiste simplement à stocker en mémoire, directement sur le process de l'API, les différents éléments.

_Ce type de cache est à utiliser pour stocker des données de référentiel en lecture seule, qui changent peu. Une considération importante à prendre en compte si vous êtes sur du multi-instances est que chaque instance aura son propre cache local (au contraire du cache distribué). Attention donc aux problématiques de synchronisations, d'invalidations, etc._

Voyons ensemble comment mettre en place un cache local sur une API ASPNET Core.

# Enregistrement du IMemoryCache

Microsoft propose l'interface `IMemoryCache`, disponible via le <a href="https://www.nuget.org/packages/Microsoft.Extensions.Caching.Memory/" title="Microsoft.Extensions.Caching.Memory Nuget Page" target="_blank">Nuget 
 Microsoft.Extensions.Caching.Memory</a>, représentant un cache mémoire et que l'on va utiliser comme cache local. L'avantage de ce Nuget est qu'il fonctionne nativement avec l'injection d'ASPNET Core.

Pour enregistrer le cache mémoire sur votre API, il suffit d'ajouter la ligne suivant dans la méthode `ConfigureServices` de la classe `Startup`: 

```csharp
 services.AddMemoryCache();
```

# Utilisation du cache mémoire

A partir de là, vous pouvez injecter l'interface <a href="https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.caching.memory.imemorycache?view=aspnetcore-2.2" title="imemorycache documentation" target="_blank">`IMemoryCache`</a> n'importe où afin d'accéder à votre cache mémoire. L'utilisation se fait alors très simplement, en appelant par exemple la méthode <a href="https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.caching.memory.cacheextensions.getorcreateasync?view=aspnetcore-2.2#Microsoft_Extensions_Caching_Memory_CacheExtensions_GetOrCreateAsync__1_Microsoft_Extensions_Caching_Memory_IMemoryCache_System_Object_System_Func_Microsoft_Extensions_Caching_Memory_ICacheEntry_System_Threading_Tasks_Task___0___" title="getorcreateasync documentation" target="_blank">`GetOrCreateAsync`</a> :

```csharp
public class MyController 
{
  private readonly IMemoryCache memoryCache;

  public MyController(IMemoryCache memoryCache)
  {
    this.memoryCache = memoryCache;
  }

  [HttpGet]
  public async Task<ActionResult> GetData()
  {
    var data = await this.memoryCache.GetOrCreateAsync("mycachekey", entry => {
      entry.SlidingExpiration = TimeSpan.FromMinutes(30);
      return [...];
    });

    return this.Ok(data);
  }
}
```

Si la clef de cache existe, la valeur stockée sera retournée (évitant ainsi un appel à la base de données), sinon la fonction passée en paramètre sera appelée, et la valeur retournée sera stockée en cache pour la durée configurée (30min dans l'exemple précédent).

Et voilà rien de plus compliqué. Nous verrons dans deux prochains articles comment hydrater le cache au démarrage de l'API et comment le mettre à jour avant son expiration pour garantir des données fraîches tout le temps.

Bonne mise en cache !
