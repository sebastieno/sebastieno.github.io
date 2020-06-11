---
layout: post
title: 'NgInspector for VorlonJS'
date: 2015-06-26
categories: javascript
resume: 'VorlonJS permet de débugger simplement des applications web à distance. Découvrons l''utilisation du plugin ''NgInspector'' que j''ai développé avec Pierre-Alexandre G. (article Anglais).'
tags: vorlonjs nginspector angularjs
---
Recently, some of Microsoft guys (some frenchies in addition) released a new tool for web developers: VorlonJS. In a few words, VorlonJS is _"an open source, extensible, platform-agnostic tool for remotely debugging and testing your JavaScript"_. You can find more information directly on its website : <a href="http://vorlonjs.com/" target="_blank">http://vorlonjs.com/</a> or here <a href="http://blogs.msdn.com/b/eternalcoding/archive/2015/04/30/why-we-made-vorlon-js-and-how-to-use-it-to-debug-your-javascript-remotely.aspx" target="_blank">Why we made vorlon.js and how to use it to debug your JavaScript remotely</a>.

![VorlonJS logo](https://sebastienollivier.blob.core.windows.net/blog/nginspector/nginspector-vorlon.jpg =368x156 "VorlonJS logo")

Because we love this kind of tool, that help us everyday by simplify difficult tasks, and because we love AngularJS too, <a href="https://twitter.com/pagury" target="_blank">@PaGury</a> and I (<a href="https://twitter.com/sebastienoll" target="_blank">@SebastienOll</a>) decided to participate to this project by creating a new plugin : NgInspector.

The idea behind this plugin is to be able to easily visualize the scopes hierarchy of an angular application and inspect scopes content.
 
## How to use it ?
 
You can see here <a href="http://vorlonjs.com/#getting-started" target="_blank">http://vorlonjs.com/#getting-started</a> how to launch VorlonJS in your local environment.

Before starting to debug, just ensure that the `config.json` file in the _Server_ folder has the enabled property to `true` for the NgInspector (without it, the plugin will not be loaded) :

```javascript
{
    "includeSocketIO": true,
    "useSSL": true,
    "SSLkey": "cert/server.key",
    "SSLcert": "cert/server.crt",
    "plugins": [
        { "id": "CONSOLE", "name": "Interactive Console", "panel": "bottom", "foldername" : "interactiveConsole", "enabled": true},
        { "id": "DOM", "name": "Dom Explorer", "panel": "top", "foldername" : "domExplorer", "enabled": true },
        { "id": "MODERNIZR", "name": "Modernizr","panel": "bottom", "foldername" : "modernizrReport", "enabled": true },
        { "id" : "OBJEXPLORER", "name" : "Obj. Explorer","panel": "top", "foldername" : "objectExplorer", "enabled": true },
        { "id" : "XHRPANEL", "name" : "XHR","panel": "top", "foldername" : "xhrPanel", "enabled": true  },
        { "id" : "NGINSPECTOR", "name" : "ngInspector","panel": "top", "foldername" : "ngInspector", "enabled": true }
    ]
}
```

When VorlonJS is launched, you can access to the dashboard and select your application in the client list. After that, you will see the NgInspector tab on the main panel.

![NgInspector tab](https://sebastienollivier.blob.core.windows.net/blog/nginspector/nginspector-tab.jpg =630x274 "NgInspector tab")

The plugin will communicate with your application to display, on its left panel, a hierarchy of angular scopes. Each scope will be associated with an icon which represents its type (root scope, scope associated to a controller, scope created by a ng-repeat instance, etc.).

![Hierarchy view](https://sebastienollivier.blob.core.windows.net/blog/nginspector/nginspector-hierarchy.jpg =626x268 "Hierarchy view")

When you click on a scope node of the hierarchy, its detail will be displayed on the right panel. Scope property nodes will be associated with an icon representing its type (object, string, function, boolean, etc.).

![Detail view](https://sebastienollivier.blob.core.windows.net/blog/nginspector/nginspector-detail.jpg =660x216 "Detail view")
 
To be able to follow your application changes, NgInspector will subscribe to those changes and will refresh its panels in consequence.

![NgInspector in action](https://sebastienollivier.blob.core.windows.net/blog/nginspector/nginspector.gif =369x300 "NgInspector in action")
 
## What's next ?
 
This version is the first version of the NgInspector plugin and we have a lot of great ideas to improve it (edit a scope property value directly from the dashboard, select a node on the DOM Explorer and show the detail of the linked scope, etc.).
 
In parallel, the core VorlonJS team and the contributors are working to add new features and new plugins.
 
If you want to be involved, you can fork the Github: <a href="https://github.com/MicrosoftDX/Vorlonjs/" target="_blank">https://github.com/MicrosoftDX/Vorlonjs/</a>. Otherwise, you can give us feedbacks, improvement ideas, etc.

 
Good debugging session !


