---
layout: post
title: 'Déploiement automatique dans Azure d''une application Angular avec VSTS'
date: 2017-08-16
categories: javascript
resume: 'Utilisons VSTS et Azure afin de créer un process de déploiement automatique d''une application Angular.'
tags: azure build angular vsts release
---
Même lorsque l'on sort des technologies Microsoft, VSTS peut s'avérer être un allié de poids dans la réussite de vos projets.

Voyons aujourd'hui comment profiter de cet outil pour déployer automatiquement, à chaque modification de code, une application Angular sur Azure.

## Création d'une application Angular avec la CLI

Pour la suite de l'article, on va partir sur une application Angular simple, créée via la CLI.

Si vous ne savez pas comment faire, c'est très rapide. Il faut d'abord installer la CLI d'Angular sur votre machine via la commande npm suivante (il vous faudra NodeJS d'installé évidemment) :

```
npm install -g @angular/cli
```

Une fois installée, il suffit d'appeler la commande new de la CLI :

```
ng new ngtoazureviavsts
```
Et voilà, notre application est prête !

## Création du hosting NodeJS avec Express
Dans Azure, notre application sera hébergée sur un serveur NodeJS. Il nous faut donc créer un serveur HTTP en NodeJS, ce que l'on va faire en utilisant le paquet npm <a title="express" href="https://www.npmjs.com/package/express" target="_blank">express</a>.

Dans le dossier /src de notre application, nous allons créer un répertoire /server qui contiendra un fichier server.js, permettant de créer le serveur HTTP, et un fichier package.json, déclarant les dépendances de notre serveur HTTP.

<img src="https://sebastienollivier.blob.core.windows.net/blog/deploiement-automatique-dans-azure-d-une-application-angular-avec-vsts/solution-explorer.png" alt="solution-explorer.png" width="168" height="226" />

Le fichier package.json déclare simplement une dépendance vers express : 

```javascript
{
  "name": "blog",
  "version": "0.0.0",
  "license": "MIT",
  "private": true,
  "dependencies": {
    "express": "^4.15.3"
  }
}
```

Le fichier server.js, quant à lui, contient le code suivant : 

```javascript
const express = require('express');
const path = require('path');
const http = require('http');

const app = express();

app.use(express.static(__dirname));
app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'index.html'));
});

const port = process.env.PORT || '3000';
app.set('port', port);

const server = http.createServer(app);
server.listen(port, () => console.log(`server running on localhost:${port}`));
```

Ce code crée simplement un serveur express qui retourne systématiquement le fichier index.html, sauf pour les fichiers statiques (je vous laisse vous référer à la <a title="Documentation express" href="https://expressjs.com/" target="_blank">documentation d'express</a> pour plus de détails sur le fonctionnement).

Il faut maintenant modifier le fichier .angular-cli.json pour intégrer les deux fichiers créés précédemment comme output de la build :

```javascript
"apps": [{
    "root": "src",
    "outDir": "dist",
    "assets": [
        "assets",
        "favicon.ico",
        {
          "glob": "server.js",
          "input": "./server/",
          "output": "./"
        },
        {
          "glob": "package.json",
          "input": "./server/",
          "output": "./"
        }
    ]
}]
```

Si vous faites un `ng build`, vous devriez maintenant avoir le contenu suivant dans le dossier dist :

![Arborescence répertoire dist](https://sebastienollivier.blob.core.windows.net/blog/deploiement-automatique-dans-azure-d-une-application-angular-avec-vsts/solution-explorer-distfolder.png =190x335 "Arborescence répertoire dist")

En exécutant le fichier server.js, vous lancerez le serveur express hostant notre application Angular :

![serverjs running](https://sebastienollivier.blob.core.windows.net/blog/deploiement-automatique-dans-azure-d-une-application-angular-avec-vsts/serverjs-running.png =429x212 "serverjs running")

Notre hosting NodeJS est prêt.

## Création de la build
Sur votre souscription VSTS, créez une nouvelle build. Nous utiliserons un template vide. La build va être composée des tâches suivantes :

* _Get sources_ : Cette tâche est créée par défaut et permet de récupérer les sources depuis plusieurs providers (VSTS via Git ou TFVC, Github, etc.). Configurez la de manière à accéder à votre repository cible.
* _npm install @angular/cli -g_ : Cette tâche de type _npm_ permet d'installer la CLI d'Angular en global, que nous utiliserons dans la suite de la build
* _npm install _: Cette tâche de type _npm_ permet d'installer toutes les dépendances npm de notre projet
* _ng build --prod --aot_ : Cette tâche de type _command line_ permet de build l'application en mode production via la CLI, en précisant le flag aot pour activer la compilation Ahead of Time
* _npm install_ (dans le répertoire dist) : Cette tâche de type _npm_ permet d'installer toutes les dépendances de notre serveur express. Attention de bien préciser dans le paramètrage que le répertoire d'exécution de la commande doit être le répertoire /dist
* _Publish Build Artifacts_ : Cette tâche de type _Publish Build Artifacts_ va publier le répertoire /dist en tant qu'artefact de la build.

La build est prête. Après exécution, vous devriez obtenir les artefacts suivant (dans mon cas, j'ai nommé l'artefact _front-app_) :

![Artéfacts de la build](https://sebastienollivier.blob.core.windows.net/blog/deploiement-automatique-dans-azure-d-une-application-angular-avec-vsts/build-artifacts.png =771x495 "Artéfacts de la build")

Dans son onglet _Triggers_, vous pouvez indiquer à la build de s'exécuter après chaque changement de code (c'est-à-dire après chaque push ou check-in) en cochant _Continuous Integration._

<h3>Et maintenant les tests !</h3>
Comme nous sommes des développeurs consciencieux, notre application est testée ! Nous allons donc rajouter l'exécution de ces tests dans le process de build afin de nous assurer de la qualité de l'application.

Par défaut, l'application créée par la CLI d'Angular va exécuter les tests en lançant un navigateur Chrome, via la commande _ng test_. Dans le cas d'une build, les tests s'exécutent via un agent sur lequel n'est pas installé Chrome, ce qui provoquera un échec. Pour éviter ce comportement, nous allons devoir exécuter nos tests sur <a title="phantomjs" href="http://phantomjs.org/" target="_blank">phantomjs</a>, un navigateur Web sans interface graphique et basé sur WebKit.

L'utilisation de ce navigateur se fait en installant deux paquets npm dans l'application, correspondant respectivement à phantomjs et au plugin Karma permettant de lancer des tests sur phantomjs :

```
npm install phantomjs-prebuilt --save-dev
npm install karma-phantomjs-launcher --save-dev
```

La configuration Karma de notre application, via le fichier `karma.config.js`, doit être modifiée afin de rajouter le plugin `karma-phantomjs-launcher` :

```javascript
plugins: [
    ...
    require('karma-phantomjs-launcher')
]
```

Vous devriez maintenant pouvoir lancer vos tests avec phantomjs via la commande `ng test --browsers=PhantomJS`.

![Exécution de karma](https://sebastienollivier.blob.core.windows.net/blog/deploiement-automatique-dans-azure-d-une-application-angular-avec-vsts/karma-execution.png =962x152 "Exécution de karma")

Nous allons également faire en sorte de pouvoir récupérer le résultat de l'exécution des tests pour l'afficher ensuite dans VSTS (ce serait dommage de voir que les tests échouent sans savoir précisemment lesquels et pour quelles raisons). On va installer maintenant un reporter Karma permettant d'exposer l'output au format JUnit :

```
npm install karma-junit-reporter --save-dev
```
Il faut, comme précédemment, déclarer ce plugin dans la configuration Karma :

```javascript
plugins: [
    ...
    require('karma-phantomjs-launcher'),
    require('karma-junit-reporter')
]
```
On va également y rajouter une propriété _junitReporter_ afin de configurer ce reporter (l'objectif étant de définir le nom du fichier qui contiendra l'output des tests).

```javascript
junitReporter: {
    outputDir: '',
    outputFile: 'test-junit.xml'
}
```

Il reste maintenant à modifier la propriété `reporters` pour ajouter JUnit :

```javascript
reporters: config.angularCli && config.angularCli.codeCoverage ? [... , 'junit'] : [... , 'junit']
```

Notre application est maintenant configurée pour que les tests puissent être exécutés sur VSTS, via phantomjs. De retour sur la build, rajoutons une tâche de type _command line_ lançant la commande suivante : `ng test --watch=false --reporters=junit,progress --browsers=PhantomJS --singleRun=true`. Les paramètres permettent  :

* _--watch=false_ : De ne pas écouter les fichiers pour relancer les tests après une modification
* _--reporters=junit,progress _: D'utiliser les reporters _junit_ et _progress_
* _--browsers=PhantomJS_ : D'utiliser le navigateur _PhantomJS_
* _--singleRun=true_ __: D'exécuter les tests une fois puis d'arrêter l'exécution

A l'heure où j'écris cet article, il existe un bug sur la capture de PhantomJS par Karma lors de l'exécution sur un agent VSTS (je vous laisse lire l'issue suivante pour plus d'informations : <a title="https://github.com/Microsoft/vsts-tasks/issues/1486" href="https://github.com/Microsoft/vsts-tasks/issues/1486" target="_blank">https://github.com/Microsoft/vsts-tasks/issues/1486</a>). Pour régler ce problème, il suffit d'ajouter une variable d'environnement (onglets _Variables_ sur la définition de la build) nommée _PHANTOMJS_BIN_ et ayant comme valeur _C:\NPM\Modules\PhantomJS.cmd._

L'exécution des tests se fait maintenant correctement, il reste à exposer le résultat des tests en ajoutant une tâche _Publish Test Results_. Sur la configuration de cette tâche, le format doit être _JUnit_ et le pattern des fichiers doit être _**/test-junit.xml_ (en fonction de la configuration Karma de votre reporter JUnit).

La build est maintenant finie et devrait ressembler à ça :

![Etapes de la build](https://sebastienollivier.blob.core.windows.net/blog/deploiement-automatique-dans-azure-d-une-application-angular-avec-vsts/build-process.png =343x504 "Etapes de la build")

Le résultat de l'exécution de la build devrait donner ça :

![Résultat de la build](https://sebastienollivier.blob.core.windows.net/blog/deploiement-automatique-dans-azure-d-une-application-angular-avec-vsts/build-output.png =1039x521 "Résultat de la build")


Il est possible de voir le détail de l'exécution de tests via l'onglet _Tests_ :

![Résultat des tests de la build](https://sebastienollivier.blob.core.windows.net/blog/deploiement-automatique-dans-azure-d-une-application-angular-avec-vsts/build-tests-output.png =1016x547 "Résultat des tests de la build")

## Création de la release
La build terminée, il reste à créer la release qui va déployer l'application sur un App Service Azure. Vous pouvez pour cela utiliser le template _Azure App Service Deployment_.

Sur la tâche _Azure App Service Deploy_, vous aurez besoin de sélectionner la souscription à utiliser ainsi que le nom de l'App Service sur laquelle notre application sera déployée. Sur le paramètre _Package or folder_, il faudra renseigner l'artefact généré par la build (dans mon cas _$(System.DefaultWorkingDirectory)/**/front-app_). Petite subtilité, comme nous souhaitons exécuter notre application sur NodeJS, nous allons demander la génération d'un fichier Web.config et y déclarer un handler _iisnode_ démarrant sur le fichier _server.js_. La release ressemble alors à ça :

![Etapes de la release](https://sebastienollivier.blob.core.windows.net/blog/deploiement-automatique-dans-azure-d-une-application-angular-avec-vsts/release.png =1422x562 "Etapes de la release")

Dans son onglet _Triggers_, vous pouvez indiquer à la release de s'exécuter après chaque build, en cochant _Continuous Deployment_ puis en sélectionnant la build. Et voilà, le déploiement continu est prêt, il ne vous reste plus qu'à coder !

## Bonus : Optimisation de la build
Si vous avez lancé la build, vous vous êtes sans doute rendu compte qu'elle est... très longue. Un peu moins de 4 minutes pour n'exécuter que quelques tests, ca fait beaucoup. Le problème vient majoritairement du fait que l'on demande l'installation d'Angular CLI en global, pour ensuite appeler les commandes ng.

Pour éviter cette installation, on va déclarer des nouvelles commandes dans le fichier package.json qui seront appelées depuis la build (de cette manière on utilisera la CLI du projet et non plus celle globale) :

```javascript
"scripts": {
    "ng": "ng",
    "start": "ng serve",
    "build": "ng build",
    "test": "ng test",
    "lint": "ng lint",
    "e2e": "ng e2e",
    "test-build": "ng test --watch=false --reporters=junit,progress --browsers=PhantomJS --singleRun=true",
    "build-prod": "ng build --prod --aot"
}
```
On peut alors supprimer l'installation de la CLI Angular, l'appel à _ng test_ et _ng build_, et remplacer tout ça par un appel aux deux commandes précédentes :

![Etapes de la build optimisée](https://sebastienollivier.blob.core.windows.net/blog/deploiement-automatique-dans-azure-d-une-application-angular-avec-vsts/build-process-optimized.png =337x449 "Etapes de la build optimisée")

Si on réexécute la build, elle ne prend maintenant "plus que" 2,5 minutes.

_Pour plus d'informations sur le framework Angular, vous pouvez vous référer à notre livre : <a href="https://www.editions-eni.fr/livre/angular-developpez-vos-applications-web-avec-le-framework-javascript-de-google-9782409008979" target="_blank">Angular: developpez vos applications web avec le framework javascript de google</a>._

Bons déploiements !

