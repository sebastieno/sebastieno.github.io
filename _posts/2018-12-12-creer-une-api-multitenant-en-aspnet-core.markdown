---
layout: post
title: 'Créer une API multi-tenant en ASPNET Core'
date: 2018-12-12
categories: asp-net-mvc
resume: 'Comment peut-on créer une API multi-tenant en ASPNET Core ?'
tags: aspnetcore multitenant
---
Dans certains types de services, il est fréquent de vouloir exposer son application en mode multi-tenant, c'est à dire avoir une même instance de son application pour plusieurs clients.

Voici la définition <a href="https://fr.wikipedia.org/wiki/Multi-tenant" target="_blank">Wikipédia</a> de la notion de multi-tenant :

_En informatique, multi-tenant, ou multi-entité désigne un principe d'architecture logicielle permettant à un logiciel de servir plusieurs organisations clientes (tenant en anglais, ou locataire en français) à partir d'une seule installation. Elle s'oppose à une architecture multi-instance où chaque organisation cliente a sa propre instance d'installation logicielle (et/ou matérielle). Avec une architecture multi-tenant, un logiciel est conçu pour partitionner virtuellement ses données et sa configuration, et chaque organisation cliente travaille avec une instance virtuelle adaptée à ses besoins._

Voyons ensemble comment mettre en place une API multi-tenant en ASPNET Core.

# Détecter le tenant via un middleware

On va commencer par créer un middleware ASPNET Core dont le rôle sera d'identifier le tenant ciblé par la requête HTTP. L'identification d'un tenant va se faire via deux critères (vous pouvez étendre ce point en fonction de votre contexte métier) : via le _Referer HTTP_ (permettant d'identifier le domaine ayant appelé notre API) ou via un header HTTP (dans le cas d'un appel d'une application non web).

```csharp
public class MultiTenantMiddleware
{
  private readonly RequestDelegate next;
  private const string MultiTenantKeyHeaderName = "Tenant-Key";

  public MultiTenantMiddleware(RequestDelegate next)
  {
      this.next = next;
  }

  public async Task Invoke(HttpContext context)
  {
    var query = context.RequestServices.GetService<GetTenantQuery>();

    if (context.Request.Headers.ContainsKey(MultiTenantKeyHeaderName))
    {
      var tenantKey = context.Request.Headers[MultiTenantKeyHeaderName];

      var tenant = await query.ByKey(tenantKey).ExecuteAsync();
      if (tenant == null)
      {
        throw new SecurityException($"The tenant with the key {tenantKey} was not found");
      }

      context.Items["Tenant"] = tenant;
    }
    else if (context.Request.Headers.ContainsKey("Referer"))
    {
      string referer = context.Request.Headers["Referer"];

      var refererUri = new Uri(referer);
      string refererHost = refererUri.Host;

      var tenant = await query.ByReferer(refererHost).ExecuteAsync();
      if (tenant == null)
      {
        throw new SecurityException($"The tenant with referer {refererHost} was not found");
      }

      context.Items["Tenant"] = tenant;
    }
    else
    {
      throw new SecurityException($"No tenant has been detected for this request");
    }

    await this.next.Invoke(context);
  }
}
```

Le code est plutôt simple. Le middleware va regarder si un header d'identification a été fourni, ou à default va prendre le header _Referer_, puis va essayer d'identifier quel tenant est associé à cette information. Vous pouvez par exemple stocker la liste des tenants dans une base de données (avec pour chaque, le ou les DNS autorisés ainsi qu'une clef d'identification). Une fois détecté (dans le cas contraire on renvoie une 401), il ne reste plus qu'à stocker les informations du tenant dans le _HttpContext_.

Il faut également bien penser à exclure toutes les URL non concernées par le multi-tenant (Swagger par exemple) :

```csharp
public async Task Invoke(HttpContext context)
{
  if (!context.Request.Path.StartsWithSegments(new PathString("/swagger"), StringComparison.InvariantCultureIgnoreCase))
  {
    [...]
  }

  await this.next.Invoke(context);
}
```

Ce middleware doit être positionné assez tôt dans la pipeline pour contextualiser assez vite la requête (mais après le middleware de gestion d'exceptions).

```csharp
app.UseMiddleware<MultiTenantMiddleware>();
```

# Utiliser le conteneur d'IoC pour contextualiser les instances

L'identification ayant été effectuée, il faut maintenant contextualiser toutes les instances de nos classes à l'opérateur courant. Pour cela, on va se baser sur le conteneur d'IoC en allant regarder l'opérateur courant dans le _HttpContext_.

Voici un exemple de résolution d'une classe dépendant d'une chaîne de connexion lié à l'opérateur :

```csharp
services.AddScoped<DbContext>(p =>
{
  var context = p.GetRequiredService<IHttpContextAccessor>();
  var tenant = context.Items["Tenant"] as Tenant

  return new DbContext(tenant.ConnectionString);
});
```

De cette manière, dès que l'on aura une dépendance vers cette classe, le conteneur la résoudra contextualisé à l'opérateur courant.

Bonnes APIs multi-tenant !
