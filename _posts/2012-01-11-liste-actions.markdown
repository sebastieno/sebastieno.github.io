---
layout: post
title: 'Accéder à la liste des actions d''un Controller depuis Javascript'
date: 2012-01-11
categories: asp-net-mvc
resume: 'Voici une solution permettant d''exposer la liste des actions d''un Controller ASP.NET MVC afin d''y accéder depuis des fichiers JavaScript.'
tags: aspnet javascript actions
---
**[Update : La solution mise en place ci-après est maintenant disponible sur GitHub et sous la forme d'un package Nuget. Pour plus d'informations : <a href="https://sebastienollivier.fr/blog/asp-net-mvc/javascriptmvcroutehelper-github-nuget/" target="_blank">https://sebastienollivier.fr/blog/asp-net-mvc/javascriptmvcroutehelper-github-nuget/</a>]**

L’une des nouveautés de ASP.NET MVC 3 est le support du <a href="http://fr.wikipedia.org/wiki/Javascript_discret" target="_blank">Unobtrusive Javascript</a> (Javascript discret en version française), c’est-à-dire le fait de ne plus embarquer de Javascript directement dans les pages web mais de les mettre dans des fichiers séparés.

L’un des problèmes liés à la séparation des pages web et du Javascript est de ne pas avoir accès aux Helper MVC dans les fichiers JS permettant notamment de générer des Url vers une action en fonction des routes déclarées : `Url.Action("Index")`. La seule solution consiste à mettre en dur les URL dans le Javascript mais on perd l’aspect dynamique proposé par les routes MVC.

_Ce post se base sur celui de <a href="http://haacked.com/archive/2011/08/18/calling-asp-net-mvc-action-methods-from-javascript.aspx?utm_source=feedburner&utm_medium=feed&utm_campaign=Feed%3A+haacked+%28you%27ve+been+HAACKED%29" target="_blank">Phil Haack</a> qui proposait comme solution à cette problématique de générer dynamiquement un fichier JS contenant un helper permettant d’appeler une action en utilisant une URL sous la forme `/json/{controller}?invoke&action={action}`. De cette manière, il n’est plus nécessaire de mettre en dur dans le fichier JS l’URL de l’action, on peut à la place utiliser son helper en fournissant le nom du controller et le nom de l’action._

Le petit bémol de cette solution est qu'elle n'utilise pas d'URL "MVC compliant". Je propose ici une modification de son implémentation pour ne plus générer un helper mais renvoyer l’URL liée à chaque action d’un controller sous la forme d’un dictionnaire Javascript. De cette manière, on pourra utiliser ce dictionnaire dans les fichiers JS pour faire référence à l’action en respectant les routes déclarées.

## Custom Route, Custom Controller et Custom Action Invoker

La première étape de la solution est de créer une Custom Route qui permettra de différencier les routes "classiques" de notre route.

```csharp
public class DiscoverableRoute : Route
{
    public DiscoverableRoute(string url)
        : base(url, new MvcRouteHandler())
    {
        if (url.IndexOf("{action}",
                StringComparison.OrdinalIgnoreCase) > -1)
        {
            throw new ArgumentException("url");
        }
    }

    public override VirtualPathData GetVirtualPath
                (RequestContext requestContext,
                 RouteValueDictionary values)
    {
        return null;
    }
}
```

Il faut ensuite créer un Custom Controller. Son objectif est de court circuiter le fonctionnement standard des controller MVC (qui appelleraient l’action demandée dans la route via le `ControllerActionInvoker`) et d’appeler un custom `ControllerActionInvoker`.

```csharp
public class DiscoverableController : Controller
{
    protected override IActionInvoker CreateActionInvoker()
    {
        return new DiscoverableControllerActionInvoker();
    }

    protected override void ExecuteCore()
    {
        if (ControllerContext.RouteData.Route is DiscoverableRoute)
        {
            ActionInvoker.InvokeAction(ControllerContext,
                    "DiscoverControllerActions");
        }
        else
        {
            base.ExecuteCore();
        }
    }
}
```

Il reste alors à surcharger le `ControllerActionInvoker` (chargé d’exécuter l’action du controller demandée) et de créer le script JS.

```csharp
public class DiscoverableControllerActionInvoker : ControllerActionInvoker
{
    public override bool InvokeAction(ControllerContext controllerContext,
                string actionName)
    {
        if (controllerContext.RouteData.Route is DiscoverableRoute
                && actionName == "DiscoverControllerActions")
        {
            return RenderJavaScriptProxyScript(controllerContext);
        }

        return base.InvokeAction(controllerContext, actionName);
    }

    private bool RenderJavaScriptProxyScript
                (ControllerContext controllerContext)
    {
        ControllerDescriptor controllerDescriptor =
                GetControllerDescriptor(controllerContext);
        ActionDescriptor[] actions =
                controllerDescriptor.GetCanonicalActions();
        Dictionary<string, string> urls =
                new Dictionary<string, string>();

        foreach (ActionDescriptor action in actions)
        {
            if (!urls.ContainsKey(action.ActionName))
            {
                string actionUrl = RouteTable.Routes.GetVirtualPath(
                    controllerContext.HttpContext.Request.RequestContext,
                    new RouteValueDictionary(
                    new {
                        controller = controllerDescriptor.ControllerName,
                        action = action.ActionName
                    })).VirtualPath;

                urls.Add(action.ActionName, actionUrl);
            }
        }

        var serializer = new JavaScriptSerializer();
        var actionMethodNames = serializer.Serialize(urls);

        string proxyScript = @"if (typeof $mvc === 'undefined')
{% raw %}{{ $mvc = {{}}; }}{% endraw %}
$mvc.{0} = {1};";
        proxyScript = String.Format(proxyScript,
                controllerDescriptor.ControllerName, actionMethodNames);

        controllerContext.HttpContext.Response.ContentType =
                "text/javascript";
        controllerContext.HttpContext.Response.Write(proxyScript);

        return true;
    }
}
```

## Utilisation

Pour utiliser tout cela, rien de plus simple. Il faut déclarer une route en utilisant notre surcharge (la route ne doit pas utiliser d'actions).

```csharp
routes.Add(new DiscoverableRoute("discoverableroute/{controller}"));
```

Il faut ensuite modifier les controller, que l’on souhaite rendre "découvrables" via ce mécanisme, pour implémenter `DiscoverableController` au lieu de `Controller`.

```csharp
public class AccountController : DiscoverableController
```

Dès que l’on souhaite récupérer la liste des actions d’un controller en Javascript, il faut ajouter la balise HTML `script` en utilisant la route déclarée précédemment.

```js
<script src="/discoverableroute/account" />
```

Le script généré est le suivant :

![Script généré](https://farm9.staticflickr.com/8345/8211405636_0d28e9bb9f_o.png =692x116 "Script généré")

Si l’on veut utiliser l’URL d’une action du controller dans un fichier Javascript, il faudra utiliser la syntaxe suivante pour l’action `LogOn` du controller `Account` :

```js
$mvc.Account.LogOn
```

Si on modifie la route déclarée par défaut par MVC en utilisant la structure `MaNouvelleRoute/{controller}/{action}/{id}`, le script généré sera le suivant :

![Script généré](https://farm9.staticflickr.com/8484/8210317221_a9148efdea_o.png =690x113 "Script généré")

J’espère que cette astuce vous sera utile.

**[Update : La solution mise en place ci-dessus est maintenant disponible sur GitHub et sous la forme d'un package Nuget. Pour plus d'informations : <a href="https://sebastienollivier.fr/blog/asp-net-mvc/javascriptmvcroutehelper-github-nuget/" target="_blank">https://sebastienollivier.fr/blog/asp-net-mvc/javascriptmvcroutehelper-github-nuget/</a>]**

A bientôt.



