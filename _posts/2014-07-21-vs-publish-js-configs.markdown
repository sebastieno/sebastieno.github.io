---
layout: post
title: 'Comment publier des fichiers de configurations JavaScript depuis Visual Studio ?'
date: 2014-07-21
categories: javascript
resume: 'Le WebDeploy permet de déployer très facilement des applications depuis Visual Studio et permet de profiter du Web.config transform. Mais comment peut-on faire dans le cas d''une SPA ?'
tags: visual-studio javascript transform
---
Depuis Visual Studio 2012, on peut profiter de la fonctionnalité de <a href="http://msdn.microsoft.com/en-us/library/dd465337(v=vs.110).aspx" target="_blank">publication d'un site Web</a>, directement depuis l'IDE, en faisant simplement clic droit sur un projet Web puis Publish...

![Visual Studio Publish Command](https://sebastienollivier.blob.core.windows.net/blog/vs-publish-javascript/vs-publish-command.png =424x224 "Visual Studio Publish Command")

De cette manière, on peut déployer un projet Web directement sur un IIS, sur un système de fichier, sur un FTP,  etc., en quelques clics. L'un des éléments très appréciable de cette fonctionnalité est de pouvoir lier un profil de publication, représentant la configuration d'une publication, à une configuration Visual Studio (Debug, Release, etc.). Cela permet notamment de profiter du mécanisme de <a href="http://msdn.microsoft.com/en-us/library/dd465326(v=vs.110).aspx" target="_blank">transformation du fichier web.config</a> pour déployer un fichier web.config différent en fonction du profil de publication.

Tout cela fonctionne très bien quand on développe une application Web classique, mais lorsque l'on développe une SPA la configuration de l'application ne se trouve plus dans le fichier web.config, situé sur le serveur, mais généralement dans un fichier JavaScript, situé côté client. Du coup, le mécanisme de transformation du web.config ne peut pas être utilisé pour mettre à jour la configuration de la SPA et donc on perd un gros avantage de la fonctionnalité de publication.

L'objectif de ce post est de montrer comment est-ce que l'on peut profiter de cette fonctionnalité dans le cadre du développement d'une application SPA.

_Cette solution ne vient pas de moi, mais de mon collègue <a href="https://twitter.com/asiffermann" target="_blank">Adrien S.</a>. Comme (malheureusement) il ne blog plus, je me suis permis (avec son accord et après relecture) de publier sa solution à sa place._

## Création de la configuration de l'application JavaScript

Pour illustrer la solution, on va partir sur une application JavaScript devant être déployée sur 3 environnements : développement pour pouvoir développer et tester en local, intégration pour pouvoir tester sur un environnement cible et production correspondant à l'environnement accessible par les utilisateurs. Pour cibler chaque environnement, avec une configuration différente, l'application va posséder 3 fichiers de configurations, un par environnement :

![JavaScript configuration files](https://sebastienollivier.blob.core.windows.net/blog/vs-publish-javascript/configuration-js-files.png =191x74 "JavaScript configuration files")

Le fichier `configuration-dev.js` contient le code suivant :

```js
(function () {
    var config = config || {};
    config.apiUrl = "http://localhost/MonApp/api";
}());
```

Le fichier `configuration-int.js` contient le code suivant :


```js
(function () {
    var config = config || {};
    config.apiUrl = "http://srv-db/MonApp/api";
}());
```

Le fichier `configuration-prod.js` contient le code suivant :

```js
(function () {
    var config = config || {};
    config.apiUrl = "http://sebastienollivier.fr/api/MonApp";
}());
```

Chaque fichier de configuration déclare une url vers la WebApi à utiliser, en fonction de son environnement. L'objectif ici est de pouvoir débugger en local en utilisant le fichier `configuration-dev.js`, de publier en intégration en utilisant le fichier `configuration-int.js` et de publier en production en utilisant le fichier `configuration-prod.js`.

## Utilisation du bon fichier de configuration

Dans la page principale de l'application (fichier `index.html` ou `_Layout.cshtml`, en fonction des cas), on va rajouter une référence vers le script `configuration.js`.

```xml
<script type="text/javascript" src="configuration.js"></script>
```

Ce fichier n'existe pas réellement. L'idée est de créer une règle de réécriture d'URL pour, en fonction de l'environnement ciblé, pointer vers l'un des 3 fichiers créés précédemment.

Si ce n'est pas déjà fait, il faut installer le module d’URL Rewrite de IIS : <a href="http://www.iis.net/downloads/microsoft/url-rewrite" target="_blank">http://www.iis.net/downloads/microsoft/url-rewrite</a>. Puis dans le fichier `web.config`, on va créer la règle suivante : 

```xml
<rewrite>
    <rules>
        <rule name="JavaScript Configuration Rule" stopProcessing="true">
            <match url="configuration.js" />
            <conditions logicalGrouping="MatchAll">
                <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
                <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
            </conditions>
            <action type="Rewrite" url="configuration/configuration-dev.js" />
        </rule>
    </rules>
</rewrite>
```

Cette règle indique que lorsqu'une requête vers l'url `configuration.js` est effectuée, le serveur va en fait cibler l'url `configuration/configuration-dev.js`.

Lorsque l'on sera en local, cette règle sera active et nous permettra de cibler le fichier de configuration de développement.

Dans le fichier `web.integration.config`, correspondant au fichier de transformation de l'environnement d'intégration, on va créer la règle de transformation suivante :

```xml
<rewrite>
    <rules>
        <rule name="JavaScript Configuration Rule" xdt:Locator="Match(name)">
            <action type="Rewrite" url="configuration/configuration-int.js" xdt:Locator="Match(type)" xdt:Transform="SetAttributes(url)" />
        </rule>
    </rules>
</rewrite>
```

La transformation précédente va modifier l'action de la règle JavaScript Configuration Rule pour pointer vers le fichier `configuration/configuration-int.js`.

Du coup, lorsque l'on va publier l'application en intégration, la règle précédente va surcharger celle définit par défaut et le fichier de configuration ciblé sera celui d'intégration.
 
De la même manière, le fichier `web.production.config` contient la règle de transformation suivante :

```xml
<rewrite>
    <rules>
        <rule name="JavaScript Configuration Rule" xdt:Locator="Match(name)">
            <action type="Rewrite" url="configuration/configuration-prod.js" xdt:Locator="Match(type)" xdt:Transform="SetAttributes(url)" />
        </rule>
    </rules>
</rewrite>
```

Chaque environnement aura donc son fichier de configuration automatiquement ciblé en fonction du profil de publication utilisé.

## Empêcher l'accès aux fichiers de configurations

Pour le moment, les 3 fichiers de configurations étant déployés peu importe l'environnement ciblé, l'utilisateur pourra avoir accès à ces fichiers directement via son navigateur (à condition qu'il devine l'URL). Cela pose évidemment un problème de sécurité puisque l'ensemble des configurations des différents environnements est exposé.

Pour sécuriser ces fichiers et empêcher les utilisateurs d'y accéder, on va rajouter une règle dans le fichier `web.config` interdisant l'accès direct : 

```xml
<rule name="JavaScript Configuration Direct Access Abort Rule" stopProcessing="true">
    <match url="^configuration/(.+).js" />
    <action type="AbortRequest" />
</rule>
```

Et voilà, l'accès direct aux fichiers de configuration est interdit.

Bonnes publications !
