---
layout: post
title: 'Réhydrater le cache local de votre API ASPNET Core avant son expiration'
date: 2019-02-20
categories: net
resume: 'L''utilité d''un cache est de fournir des données fraîches. Comment éviter que ces données expirent et que notre cache ne soit plus à jour ?'
tags: azure alwayson inmemory cache
---
Nous avons vu dans l'article précédent <a href="https://sebastienollivier.fr/blog/asp-net-mvc/mettre-en-place-un-cache-local-sur-une-api-aspnetcore">Mettre en place un cache local sur une API ASPNET Core</a> comment mettre en place un cache mémoire local via le nuget <a href="https://www.nuget.org/packages/Microsoft.Extensions.Caching.Memory/" title="Microsoft.Extensions.Caching.Memory Nuget Page" target="_blank">Microsoft.Extensions.Caching.Memory</a> (n'hésitez pas à aller jeter un oeil à l'article si vous ne l'avez pas lu).

Grâce à ce cache, nous pouvons proposer aux utilisateurs de nos API un temps de réponse reduit et par conséquence une capacité à encaisser une charge plus importante.

Il reste par contre le moment où notre cache va expirer. A ce moment là, le ou les prochaines requêtes ne vont pas pouvoir s'y appuyer et vont devoir le réhydrater (en allant par exemple effectuer une requête en base de données), ce qui dommage...

Voyons ensemble comment anticiper l'expiration du cache pour le rafraichir en amont, en s'appuyant sur le _AlwaysOn_ d'Azure.

# Mise à jour du cache au AlwaysOn

Lorsque l'on crée un AppService Azure, en fonction du plan sélectionné, on peut choisir d'activer l'option `AlwaysOn` :

<img src='https://sebastienollivier.blob.core.windows.net/blog/utiliser-le-alwayson-azure-pour-garder-le-cache-de-votre-api-a-jour/alwayson.png' alt='Azure AlwaysOn configuration' />

Avec cette option activée, Azure va envoyer toutes les 5 min (environ) une requête vers votre API pour s'assurer que celle-ci reste toujours disponible (c'est à dire pour que l'Application Pool ne soit pas éteint). L'idée ici va être de profiter de cette requête pour réhydrater les clefs de cache qui ne sont plus valides (ou celles qui sont sur le point d'être invalidées) de manière à ce que les utilisateurs accèdent toujours à un cache frais.

## Enregistrement de la date d'expiration

Le `IMemoryCache` n'expose pas la durée de validité des éléments en cache. On ne va donc pas pouvoir simplement parcourir les clefs de cache à la recherche des éléments expirés.

Au lieu de stocker directement la valeur dans le `IMemoryCache`, on va stocker un `Tuple` contenant la valeur ainsi que sa date d'expiration :

```csharp
var data = (await this.memoryCache.GetOrCreateAsync("mycachekey", entry => {
  var expirationDate = DateTime.UtcNow + TimeSpan.FromMinutes(30);
  entry.AbsoluteExpirationRelativeToNow = expirationDate;

  return new Tuple<Data, DateTime>([...], expirationDate);
})).Item1;
```

De cette manière, à chaque fois que l'on va récupérer un élément, on aura la possibilité de connaître la durée restante pendant laquelle celui-ci est considéré comme valide par le cache.

## Détection de la requête AlwaysOn d'Azure

Lorsque Azure va pinger notre API pour le _AlwaysOn_, il va positionner le header `user-agent` à la valeur `AlwaysOn` (de manière à ce qu'on puisse l'identifier). On va donc créer un middleware qui va surveiller ce header à la recherche du AlwaysOn :

```csharp
public class AlwaysOnMiddleware
{
  private readonly RequestDelegate next;

  public AlwaysOnMiddleware(RequestDelegate next)
  {
      this.next = next;
  }

  public async Task Invoke(HttpContext context)
  {
    if (context.Request.Headers.ContainsKey("user-agent") && context.Request.Headers["user-agent"] == "AlwaysOn")
    {
      context.Response.StatusCode = (int)HttpStatusCode.OK; 

      //TODO: réhydrater le cache
    }
    else
    {
      await this.next.Invoke(context);
    }
  }
}
```

Si on trouve le header, on positionne simplement le code HTTP de la réponse à OK et on réhydrate le cache (point qu'on verra juste après). Ici, on n'appelle pas le `next.Invoke` puisqu'on ne souhaite pas que les autres middlewares traitent la requête.

## Réhydrater le cache

L'idée ici est de récupérer la valeur en cache et de regarder si elle est toujours valide ou si elle va bientôt expirer.

```csharp
if (cache.TryGetValue("mycachekey", out var result))
{
  var timedResult = result as Tuple<T, DateTime>;
  if ((timedResult.Item2 - TimeSpan.FromMinutes(gracePeriodInMinute)) < DateTime.UtcNow)
  {
    result = null;
  }
}

if(result == null)
{
  //TODO: réhydratation du cache
}
```

Le code est relativement simple. On récupère l'élement voulu. Si on réussit (c'est-à-dire qu'il n'est pas expiré), on vérifie que sa date d'expiration est dans plus que _x_ minutes. Dans les cas contraires, il faut réhydrater le cache :

```csharp
var entry = cache.CreateEntry(key);
var expirationDate = DateTime.UtcNow + TimeSpan.FromMinutes(30);

entry.SetValue(result);
entry.AbsoluteExpirationRelativeToNow = expirationTimeSpan;
entry.Dispose();

```

Le code est là encore très simple. On récupère l'entrée du cache et on met à jour sa valeur ainsi que son expiration.

## Encapsulation

Evidemment, si vous utilisez beaucoup de clef de cache, sans encapsulation, le code précédent va devenir vite imbitable. Ce que je fais généralement, c'est que je déclare une interface exposant une méthode _RefreshCache_ et implémentées par mes classes responsables de la gestion du cache (Repository, QueryCommand, etc. en fonction de votre structure). Le middleware vu précédemment va alors résoudre toutes les classes implémentant l'interface (via DI) pour déclencher les méthodes _RefreshCache_. De cette manière, on peut ajouter un nouvel élément de cache "réhydratable" sans avoir à revoir toute la chaîne :

```csharp
 var refreshableCaches = scope.ServiceProvider.GetServices<IRefreshableCacheQuery>();
 foreach (var refreshableCache in refreshableCaches)
 {
    await refreshableCache.RefreshCache(gracePeriod);
 }
```

Bonne réhydratation de cache !
