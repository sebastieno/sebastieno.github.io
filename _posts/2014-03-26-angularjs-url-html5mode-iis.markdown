---
layout: post
title: 'AngularJS, URL Html5Mode & IIS'
date: 2014-03-26
categories: angularjs
resume: 'Comment configurer IIS pour faire fonctionner une application AngularJS configurée avec le mode d''URL HTML 5 ?'
tags: angularjs html5mode iis
---
Par défaut, le mécanisme de navigation d'AngularJS se base sur des URLs de la forme : http://localhost/index.html#recherche.

Le _#recherche_ indique à AngularJS d'afficher la vue correspondant à la route _recherche_. Peu importe la route renseignée, la page active sera toujours _index.html_, correspondant à la page principale de l'application et contenant notamment la directive `ng-view`.

AngularJS propose d'utiliser des URLs plus élégantes via l'option `html5Mode` :

```javascript
module.config(function ($locationProvider) {
    $locationProvider.html5Mode(true);
});
```

De cette manière, les URLs ne seront plus composées de la page active suivie du nom de la route préfixé par #. L'URL précédente sera alors : http://localhost/recherche

Si votre application est hébergée sur IIS, pour que le mécanisme AngularJS d'URL HTML 5 fonctionne, il est nécessaire de configurer légèrement IIS.

## URL Rewrite

Lorsqu'un utilisateur va se connecter à notre application, une requête vers une URL de la forme http://&lt;hostname&gt;/&lt;routeAngularJS&gt; va être effectuée. IIS va essayer de résoudre cette URL et renvoyer un code HTTP 404 (Not Found) puisqu'il n'arrivera pas à trouver la ressource correspondant à l'URL sur le serveur. Pour corriger cela, il va falloir rediriger toutes les requêtes vers la page principale, dans notre cas _index.html_

Pour ce faire, on va devoir installer le module d'URL Rewrite de IIS : <a href="http://www.iis.net/downloads/microsoft/url-rewrite" target="_blank">http://www.iis.net/downloads/microsoft/url-rewrite</a>.

Dans le fichier `web.config` de l'application, il faut rajouter la règle suivante, qui redirige toutes les requêtes vers / : 

```xml
<rewrite>
    <rules>
        <rule name="Main Rule" stopProcessing="true">
            <match url=".*" />
            <conditions logicalGrouping="MatchAll">
                <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
                <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
             </conditions>
             <action type="Rewrite" url="/" />
        </rule>
    </rules>
</rewrite>
```

## Base URL

Maintenant que IIS renvoie vers la bonne page, il va falloir renseigner la racine du site qui sera utilisée pour résoudre les liens relatifs. Sans cela, les URLs de l'application pourraient être fausses et déclencher des erreurs HTTP 404 (si par exemple l'application est hébergée dans un répertoire virtuel via l'URL http://localhost/MonRepertoireVirtuel et que l'on crée un lien vers la route recherche de la forme `<a href="/recherche">[...]</a>`, une requête vers l'URL http://localhost/recherche sera effectuée au lieu de http://localhost/MonRepertoireVirtuel/recherche).

La balise <a href="http://www.w3.org/TR/html-markup/base.html" target="_blank">`base`</a>, à placer dans le `head` de la page, va nous permettre de définir cette information :

```html
<html>
<head>
    <meta charset="utf-8">
    <meta name="description" content="">
    <meta name="author" content="S&#233;bastien Ollivier">
    <base href="/" />

    <title>Accueil</title>
    [...]
</head>

[...]
```

Si l'application est hébergée dans un répertoire virtuel, l'URL de base sera la suivante :

```html
<base href="/MonRepertoireVirtuel/" />
```

Notre application est maintenant fonctionnelle et possède des URLs élégantes.

Bon AngularJS !

