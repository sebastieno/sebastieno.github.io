---
layout: post
title: 'Comment insérer du contenu dans un document Word 2013 depuis une App Office ?'
date: 2014-02-25
categories: app-office
resume: 'Comment intéragir avec un document Word 2013 pour y insérer des données depuis une application Office ?'
tags: app-office word
---
L'arrivée des Apps pour Office nous permet d'envisager de nouveaux scénarios d'intégration d'applications directement dans la suite Office (Excel, Word, PowerPoint ou Project).

On va voir dans cet article comment on peut intéragir avec un document Word pour y insérer des données depuis une application Office.

## Création d'une application Office

Pour créer une nouvelle App Office avec Visual Studio 2013, il suffit de créer un nouveau projet de type _App for Office 2013_.

![Template Visual Studio App for Office](https://sebastienollivier.blob.core.windows.net/blog/comment-inserer-du-contenu-dans-un-document-word-depuis-une-app-office/vs-project-template-appoffice.png =550x308 "Template Visual Studio App for Office")

Le setup permet de définir quel type d'app on souhaite créer (dans notre cas, on va garder _Task pane app in_ pour créer une application qui apparaîtra dans un panneau latéral).

![Setup Template Visual Studio App for Office](https://sebastienollivier.blob.core.windows.net/blog/comment-inserer-du-contenu-dans-un-document-word-depuis-une-app-office/vs-project-template-appoffice-setup.png =550x468 "Setup Template Visual Studio App for Office")

La solution créée va contenir 2 projets, un projet Web correspondant à l'application que l'on va créer et un projet Office qui sera chargé d'intégrer l'application Web dans Office. Le fichier `office.js` du projet Web (situé dans le répertoire `Scripts/Office/1.0/` ou accessible via le CDN <a href="https://appsforoffice.microsoft.com/lib/1.0/hosted/office.js" target="_blank">https://appsforoffice.microsoft.com/lib/1.0/hosted/office.js</a>) contient toutes les API Javascript exposées par Office. C'est avec cette API que nous allons travailler.

![Structure projet Visual Studio App For Office](https://sebastienollivier.blob.core.windows.net/blog/comment-inserer-du-contenu-dans-un-document-word-depuis-une-app-office/vs-officeapp-solution-structure.png =226x333 "Structure projet Visual Studio App For Office")

## Accès au document en Javascript

Avant de pouvoir utiliser les API Office, il va falloir attendre que le contexte du document ait eu le temps de s'initialiser. Pour cela, il faut utiliser la syntaxe suivante :

```js
Office.initialize = function () {
    // Le contexte est maintenant initialisé
}
```

Pour accéder au document courant, l'API Javascript fournit un objet <a href="http://msdn.microsoft.com/en-us/library/office/fp142295.aspx" target="_blank">`Document`</a> : 

```js
Office.context.document
```

## Insertion de contenu

L'insertion de contenu va se faire via la méthode <a href="http://msdn.microsoft.com/en-us/library/office/fp142145.aspx">`setSelectedDataAsync`</a> de l'objet `document`. Cette méthode permet d'insérer du contenu uniquement à la position du curseur (si une sélection est en cours, le contenu sera remplacé).

Le premier paramètre à passer à la méthode est le contenu à insérer. Le deuxième paramètre correspond aux options de l'insertion et va nous permettre de stipuler le type de données (via la propriété `coercionType`) que l'on va insérer (ex: texte, html, xml au format Office Open XML, etc.). Le troisième et dernier paramètre est un callback qui sera exécuté quand l'insertion aura été réalisée ou aura échouée.

Pour ajouter du texte, l'appel ressemblera à :

```js
Office.context.document.setSelectedDataAsync("Mon message inséré",
    { coercionType : "text" }, 
    function(asyncResult) {
        // Callback
    });
```

Pour ajouter du code html, l'appel ressemblera à :

```js
Office.context.document.setSelectedDataAsync("<p>Mon message <em>inséré</em></p>",
    { coercionType : "html" }, 
    function(asyncResult) {
        // Callback
    });
```

Pour ajouter un template Office Open XML, l'appel ressemblera à :

```js
Office.context.document.setSelectedDataAsync(templateOoxml,
    { coercionType : "ooxml" }, 
    function(asyncResult) {
        // Callback
    });
```

## Utilisation des bindings

Evidemment, l'insertion de contenu uniquement au niveau de la position du curseur est très limitant. Pour contourner cela, on va avoir la possibilité d'insérer du contenu dans des _Rich Text Content Control_ en passant par la fonctionnalité de binding.

### Création des _Rich Text Content Control_ dans le document Word

La création de _Rich Text Content Control_ se fait via l'onglet _Developer_ de Word. Pour l'activer, il faut aller dans le menu File > Options > Customize Ribbon et cocher _Developer_.

![Activation onglet Developper](https://sebastienollivier.blob.core.windows.net/blog/comment-inserer-du-contenu-dans-un-document-word-depuis-une-app-office/developper-tab.png =342x88 "Activation onglet Developper")

A partir de l'onglet _Developer_, on aura un bouton _Rich Text Content Control_ qui va en insérer un dans le document Word. Une fois inséré, le bouton _Properties_ va nous permettre de définir un titre au contrôle, que nous utiliserons plus tard pour créer le binding.

![Toolbox Word Rich Text Content Control](https://sebastienollivier.blob.core.windows.net/blog/comment-inserer-du-contenu-dans-un-document-word-depuis-une-app-office/richtextcontentcontrol-toolbox.png =201x91 "Toolbox Word Rich Text Content Control")

![Properties Word Rich Text Content Control](https://sebastienollivier.blob.core.windows.net/blog/comment-inserer-du-contenu-dans-un-document-word-depuis-une-app-office/richtextcontentcontrol-properties.png =493x160 "Properties Word Rich Text Content Control")

## Création du binding sur les _Rich Text Content Control_

Pour pouvoir intéragir avec le _Rich Text Content Control_ depuis l'app Office, il va falloir passer par la fonctionnalité de binding de l'API Office. La première étape va consister à créer un binding sur le contrôle en utilisant la méthode <a href="http://msdn.microsoft.com/en-us/library/office/fp123590.aspx" target="_blank">`addFromNamedItemAsync`</a> de la propriété `bindings` de l'objet `document`.

Les paramètres à passer vont être le titre du contrôle (défini lors de sa création), le type de binding (pour l'insertion de texte, ce sera "text"), des options pour notamment définir l'id du binding (qui nous sera utile pour y insérer du contenu) et une fonction de callback qui sera appelée quand le binding aura été effectué, avec succès ou non. La syntaxe sera la suivante pour créer un binding, dont l'id sera "MonBinding", sur un contrôle, dont le titre serait "MonContentControl" :

```js
Office.context.document.bindings.addFromNamedItemAsync(
    "MonContentControl", 
    "text", 
    { id: "MonBinding" }, 
    function (asyncResult) {
    });
```

Maintenant que le binding est en place, on va pouvoir utiliser la méthode <a href="http://msdn.microsoft.com/en-us/library/office/fp161004.aspx" target="_blank">`select`</a> de l'API pour le sélectionner puis la méthode <a href="http://msdn.microsoft.com/en-us/library/office/fp161120.aspx" target="_blank">`setDataAsync`</a> pour y appliquer un contenu. La méthode `select` prend un sélecteur, de la forme `bindings#{IdDuBinding}` pour sélectionner un binding via son id, ce qui donnerait dans notre exemple :

```js
Office.select("bindings#MonBinding")
```

La méthode `setDataAsync` prend en paramètre le contenu à insérer, des options permettant notamment de renseigner le type de données à insérer ainsi qu'un callback (de manière identique à la méthode `setSelectedDataAsync` vu précédemment).

Pour appliquer un contenu au binding, il faut utiliser la syntaxe suivante : 

```js
Office.select("bindings#MonBinding")
      .setDataAsync("Mon texte bindé", { coercionType: "text" });
```

### Insertion des _Rich Text Content Control_ depuis l'app Office

On a vu comment créer un _Rich Text Content Control_ directement depuis Word puis comment y appliquer un contenu depuis l'App Office, en utilisant un binding. Si le document Word ne contient pas les _Rich Text Content Control_ attendus (par exemple si l'utilisateur n'utilise pas un template Word contenant ces contrôles), la création du binding échouera et l'insertion ne pourra pas se faire.

Pour éviter cela, on peut insérer les _Rich Text Content Control_ en utilisant la méthode `setSelectedDataAsync` vue précédemment (donc malheureusement uniquement au niveau du curseur). Le seul moyen d'insérer ces contrôles est de passer par le format <a href="http://fr.wikipedia.org/wiki/Office_Open_XML" target="_blank" title="Wikipedia Ooxml">ooxml</a> en utilisant la syntaxe suivante :

```js
Office.context.document.bindings.addFromNamedItemAsync(
    "MonContentControl",
    "text",
    { id: "MonBinding" },
    function (asyncResult) {
        if(asyncResult.status == "failed" 
           && asyncResult.error.message == "The named item does not exist.") {
	       Office.context.document.setSelectedDataAsync(templateOoxml
                   , { coercionType: 'ooxml' });
        }		
    }
);
```

Dans le callback de la création du binding, si la création n'a pas pu se faire et que l'erreur indique que le _Rich Text Content Control_ "MonContentControl" n'existe pas, on insert le template ooxml (contenant ce contrôle).

On peut maintenant réappliquer les bindings et l'insertion pourra se faire avec succès :

```js,
Office.context.document.bindings.addFromNamedItemAsync(
    "MonContentControl",
    "text",
    { id: "MonBinding" },
    function (asyncResult) {
        if(asyncResult.status == "failed" 
           && asyncResult.error.message == "The named item does not exist.") {
	       Office.context.document.setSelectedDataAsync(templateOoxml
                , { coercionType: 'ooxml' }
                , function () {
                    Office.context.document.bindings.addFromNamedItemAsync(
						"MonContentControl",
						"text",
						{ id: "MonBinding" });
				});
        }
    }
);
```

### Création du template ooxml

Il nous reste maintenant à créer le template ooxml contenant le ou les _Rich Text Content Controls_. Pour cela, on va créer un nouveau document Word et y ajouter tout le contenu souhaité (contenu, _Rich Text Content Controls_, mise en page, etc.). Une fois sauvegardé, il faut dézipper le document .docx.

![Word dézippé](https://sebastienollivier.blob.core.windows.net/blog/comment-inserer-du-contenu-dans-un-document-word-depuis-une-app-office/unzipped-word.png =385x247 "Word dézippé")

Les deux fichiers qui nous intéressent sont les fichiers _document.xml_ et _styles.xml_ situés dans le dossier _word_ du document dézippé.

![Contenu d'un word dézippé](https://sebastienollivier.blob.core.windows.net/blog/comment-inserer-du-contenu-dans-un-document-word-depuis-une-app-office/unzipped-word-content.png =382x302 "Contenu d'un word dézippé")

Pour créer le template, il faut insérer le contenu de ces fichiers (sans l'entête xml) dans la structure suivante et enregistrer le fichier au format xml :

```xml
<pkg:package xmlns:pkg="http://schemas.microsoft.com/office/2006/xmlPackage">
  <pkg:part pkg:name="/_rels/.rels" pkg:contentType="application/vnd.openxmlformats-package.relationships+xml" pkg:padding="512">
    <pkg:xmlData>
      <Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships">
        <Relationship Id="rId1" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/officeDocument" Target="word/document.xml"/>
      </Relationships>
    </pkg:xmlData>
  </pkg:part>
  <pkg:part pkg:name="/word/_rels/document.xml.rels" pkg:contentType="application/vnd.openxmlformats-package.relationships+xml" pkg:padding="256">
    <pkg:xmlData>
      <Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships">
        <Relationship Id="rId1" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/styles" Target="styles.xml"/>
      </Relationships>
    </pkg:xmlData>
  </pkg:part>
  <pkg:part pkg:name="/word/document.xml" pkg:contentType="application/vnd.openxmlformats-officedocument.wordprocessingml.document.main+xml">
    <pkg:xmlData>
<!-- Insérez le xml du fichier document.xml ici :
     Ex:
      <w.document [...]>
      </w:document>
-->     
    </pkg:xmlData>
  </pkg:part>
  <pkg:part pkg:name="/word/styles.xml" pkg:contentType="application/vnd.openxmlformats-officedocument.wordprocessingml.styles+xml">
    <pkg:xmlData>
<!-- Insérez le xml du fichier styles.xml ici :
     Ex:
      <w:styles [...]>
      </w:styles>
-->
    </pkg:xmlData>
  </pkg:part>
</pkg:package>
```

Le template ooxml est maintenant prêt à être inséré.

Bonne insertion !

