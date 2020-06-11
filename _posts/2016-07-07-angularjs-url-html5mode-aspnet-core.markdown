---
layout: post
title: 'AngularJS, Url Html5Mode & ASPNET Core'
date: 2016-07-07
categories: angularjs
resume: 'Quelle configuration est nécessaire afin de faire fonctionner une application AngularJS configurée avec le mode d''URL HTML 5, hébergée dans via ASPNET Core ?'
tags: angularjs aspnet-core
---
On avait vu dans <a href="http://sebastienollivier.fr/blog/angularjs/angularjs-url-html5mode-iis">cet article (AngularJS, URL Html5Mode & IIS)</a> comment utiliser le mécanisme AngularJS d'URL HTML 5 lorsque l'application était hébergée dans IIS, notamment en définissant des règles de réécriture d'URL (via le module IIS Url Rewrite).

Avec ASPNET Core, on n'est plus nécessairement hébergé sur un IIS, donc le mécanisme précédent ne fonctionne plus systématiquement. On va voir dans cet article comment implémenter un comportement similaire en ASPNET Core.

_Si vous utilisez ASPNET Core sur un IIS, l'ancien mécanisme fonctionne toujours._

## Création d'un middleware

Avec ASPNET Core, il est possible d'agir dans la chaîne de traitement d'une requête HTTP au travers des middlewares (je vous laisse vous réferez à ce lien <a href="https://blogs.infinitesquare.com/b/wklein/archives/middleware-en-aspnet-core" target="_blank">Les middlewares en ASP.NET Core 1.0</a> pour plus d'informations).

Nous allons donc créer un middleware qui va être chargé de renvoyer la page par défaut de l'application (index.html dans la plupart des cas) lorsque la requête ne pointera vers aucune ressource.

```csharp
app.Use(async (context, next) =>
{
    await next();

    if (context.Response.StatusCode == 404 && !Path.HasExtension(context.Request.Path.Value))
    {
    	context.Request.Path = "/index.html";
        await next();
    }
});
```

L'idée ici est de regarder si le code HTTP courant est 404, ce qui signifie que l'on n'a pas trouvé de résultat à renvoyer, puis de regarder si l'url de la requête ne contient pas une extension, ce qui signifie que l'on requêterais une ressource physique (js, css, font, img, etc.). Si l'on est dans ce cas, on renvoi le contenu du fichier index.html et AngularJS prendre le relais pour naviguer vers la bonne page.

Pour que cela fonctionne, il est nécessaire d'enregistrer ce middleware avant le UseStaticPages et UseMvc :

```csharp
app.Use(...);

app.UseStaticFiles();
app.UseMvc();
```

Et voilà, notre application ASPNET Core est prête. Pour toute la configuration de l'application AngularJS, je vous laisse vous référer à l'<a href="https://sebastienollivier.fr/blog/angularjs/angularjs-url-html5mode-iis">ancien article</a>.


Bons middlewares !

