---
layout: post
title: 'Modifier la réponse HTTP dans un middleware OWIN'
date: 2017-02-28
categories: asp-net-mvc
resume: 'Comment modifier la réponse HTTP dans un middleware OWIN après l''exécution de l''action MVC ?'
tags: aspnet owin middleware
---
Si vous utilisez OWIN dans vos applications ASPNET 4, vous pouvez profiter du mécanisme des middlewares, comme avec ASPNET Core (pour plus d'informations sur les middlewares, je vous laisse vous réferer à l'article <a title="Les middlewares en ASP.NET Core 1.0" href="https://blogs.infinitesquare.com/posts/web/middleware-en-aspnet-core" target="_blank">Les middlewares en ASP.NET Core 1.0</a>).

Dans certains cas, il peut être nécessaire de modifier la réponse (le code HTTP par exemple) dans un middleware, par exemple si l'on effectue un traitement métier (log fonctionnel, etc.) après être passé dans l'action MVC et que ce traitement échoue. Le code ressemblerait alors à quelque chose comme :

```csharp
app.Use(async (context, next) => 
{
    await Next.Invoke();

    // Do business stuff
    context.Response.StatusCode = 500;
});
```

Le code précédent ne fonctionne pas correctement. Si vous le testez, vous pourrez constater que le code HTTP n'a pas été modifié. On reçoit toujours le code HTTP qui a été renvoyé par l'action MVC. La raison derrière ce comportement est qu'ASPNET MVC va commencer à renvoyer la réponse HTTP dès que l'action aura été exécutée (concrètement à l'appel de `return Request.CreateResponse(HttpStatusCode.OK)` ou équivalent). Donc notre modification du statut HTTP sera effectuée trop tard.

Il est possible d'agir lors de l'envoi des headers de la réponse en s'abonnant à l'event `OnSendingHeaders` comme ci-dessous :

```csharp
app.Use(async (context, next) => 
{
    context.Response.OnSendingHeaders(state =>
    {
       var resp = (OwinResponse)state;

       resp.StatusCode = 500;
    }, response);
    
    await Next.Invoke();

    // Do business stuff
});
```

L'inconvénient de cet évènement est que la suite de notre middleware (et des middlewares parents) n'est pas encore appelée à ce moment. On agit ici dès que la réponse a été créée (après l'appel à `return Request.CreateResponse(HttpStatusCode.OK)` ou équivalent).

Pour pouvoir modifier le contenu de la réponse directement dans notre middleware, il faut changer la référence du stream liée au body de la réponse comme ci-dessous :

```csharp
app.Use(async (context, next) => 
{ 
    var buffer = new MemoryStream();
    var body = context.Response.Body;
    context.Response.Body = buffer; 

    await next.Invoke(); 

    // Do business stuff
    context.Response.StatusCode = 500;

    buffer.Position = 0;
    await buffer.CopyToAsync(body);
});
```

L'idée est de remplacer le stream pour empêcher ASPNET d'envoyer la réponse avant d'avoir exécuter la totalité du middleware.

Cette solution a évidemment une conséquence d'un point de vue performance puisque la réponse sera envoyée plus tard.

Bon middlewares !

