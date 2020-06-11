---
layout: post
title: '1 code = 4 applications avec Cordova : Mythe ou Réalité ?'
date: 2016-04-13
categories: cordova
resume: 'Cordova promet de nous faire gagner du temps et de l''énergie en développant une seule fois notre application mais en ciblant l''ensemble des OS et devices. Qu''en est-il réellement ?'
tags: cordova
---
Depuis plusieurs années, la question du développement cross-platform reste sans réponse catégorique. Alors que le nombre de devices et d'OS ne cesse d'augmenter (iOS, Android, Windows, Linux, Mac / Phone, Phablet, Tablette, PC, Grand ecran, etc.), comment éviter de multiplier le coût de développement et de maintenance d'une application ? Comment capitaliser un maximum sur un développement en adressant le maximum d'utilisateurs (c’est-à-dire le maximum de plateformes) ?

Je vous propose dans cet article de découvrir le Framework <a href="https://cordova.apache.org/" target="_blank">Cordova</a>.

## Cordova, qu'est-ce que c'est ?

Cordova est un Framework qui propose, depuis plusieurs années, l'une des solutions les plus abouties au développement cross-platform (probablement avec Xamarin et Unity pour le domaine du jeu). L'idée derrière ce Framework est plutôt simple : si les langages Web (HTML / CSS / JS) sont connus de tous les devices / toutes les plateformes (quel device aujourd'hui n'est pas capable d'afficher une application Web ?) pourquoi ne pas les utiliser comme langages universels ?

La promesse est donc ici, à partir d'une application Web, de pouvoir créer une application native par plateforme (Android, iOS, Windows, mais aussi FirefoxOS, Blackberry et d'autres). Contrairement à Xamarin, le parti pris est ici de partager la totalité du code de l'application, du code métier à l'interface utilisateur (_il existe cependant des solutions pour avoir une IHM adaptée à chaque plateforme, comme le framework Ionic ou les merges Cordova_).

## Techniquement, comment ça marche ?

Cordova agit comme un conteneur : son rôle est d'encapsuler une application Web dans une application native, on parle alors d'application hybride. Techniquement, Cordova va embarquer les ressources de l'application Web (fichiers HTML / JS / CSS mais aussi images, fonts, etc.) dans l'application native puis les afficher à partir d'un composant WebView (customisé pour aller chercher les ressources en local plutôt que sur internet).

![Architecture d'une application Cordova](https://sebastienollivier.blob.core.windows.net/blog/1code=4applications/cordova.jpg =222x438 "Architecture d'une application Cordova")

De cette manière, l'utilisateur va naviguer sur l'application Web depuis l'application native, sans s'en rendre compte (si c'est bien fait).

Evidemment, une application Web ne peut pas accéder à autant d'informations sur le device qu'une application native (bien que les API HTML 5 deviennent de plus en plus complètes, mais encore assez mal supportées). Pour corriger cela, et donner aux applications Web autant de liberté que les applications natives, Cordova propose un mécanisme de plugins permettant à du code JavaScript d'appeler du code natif :

![Mécanisme d'un plugin Cordova](https://sebastienollivier.blob.core.windows.net/blog/qrcode-cordova/cordovaplugin-mecanism.png =392x598 "Mécanisme d'un plugin Cordova")

Un plugin Cordova est composé d'un contrat JavaScript, définissant les fonctions qui seront exposées à l'application web, et d'une implémentation par plateforme (généralement iOS, Android et Windows Phone). A la génération d'une application, Cordova injectera l'implémentation correspondant à la plateforme cible, ce qui permet d'avoir un même code JavaScript qui fonctionnera sur toutes les plateformes (en tout cas celles supportées par le plugin).

Actuellement, il existe plus de <a href="https://cordova.apache.org/plugins/" target="_blank">1000 plugins Cordova</a>, allant du "simple" gestionnaire de fichiers au composant de scan de QRCode.

Cordova a également prévu la possibilité d'injecter du code différent par plateforme lorsque l'on souhaite gérer une spécificité (ajout d'une feature, adaptation du l'UI, gestion d'une spécificité technique, etc.) via les dossiers `merges`. Le principe est ici d'ajouter les fichiers spécifiques à une plateforme dans un répertoire portant son nom, qui seront uniquement déployés sur cette plateforme.

![Dossier merges](https://sebastienollivier.blob.core.windows.net/blog/1code=4applications/merges.jpg =200x284 "Dossier merges")

## Mais est-ce que ça marche vraiment ?

Cordova souffre d'une mauvaise réputation depuis plusieurs années, notamment depuis que Facebook a annoncé en 2012 l'arrêt du développement hybride pour leurs applications au profit d'applications natives : <a href=" http://venturebeat.com/2012/09/11/facebooks-zuckerberg-the-biggest-mistake-weve-made-as-a-company-is-betting-on-html5-over-native/" target="_blank">http://venturebeat.com/2012/09/11/facebooks-zuckerberg-the-biggest-mistake-weve-made-as-a-company-is-betting-on-html5-over-native/</a>. On entend souvent parler d'applications lentes ou peu performantes, d'expérience utilisateur pas à la hauteur, d'accès limité aux informations du device, d'outillage inexistant, etc. Il y a quelques années, ces affirmations avaient du sens (je ne dis pas quelles étaient totalement vraies mais il y avait des limitations). Les choses ont énormément évolué et on se retrouve aujourd'hui avec une solution stable permettant de créer des applications hybrides de qualité identiques aux applications natives.

Le point le plus critiqué concerne les performances faibles des applications Cordova. On ne parle pas de performances pures (dans la plupart des cas, si l'application met 250ms à écrire un fichier au lieu de 200ms, ce n'est pas critique) mais plutôt de fluidité de l'application (créer des animations fluides, des transitions souples, etc.). Avant, on devait faire ces animations en JavaScript ce qui impliquait inévitablement des problèmes de performances (au moins sur les navigateurs mobiles, donc sur les applications Cordova). Depuis l'arrivée des spécifications CSS3 du W3C sur les <a href="http://caniuse.com/#feat=css-animation" target="_blank">animations</a>, on peut créer des animations directement en CSS ce qui permet d'avoir des performances beaucoup plus élevées, et donc d'avoir des animations fluides (on peut même profiter de l'accélération matérielle, par exemple en utilisant la syntaxe `transform: translate3d(0,0,0);`). Ce point change fondamentalement la qualité des applications Cordova puisqu'il permet de fournir une sensation de réactivité et de fluidité à l'utilisateur.<br>Les performances pures ont également été améliorées depuis quelques années avec des navigateurs de plus en plus performants (moteur de rendu HTML, moteur d'exécution JavaScript, etc.), ce dont bénéficient directement les applications Cordova.

Concernant l'interaction avec le device, là aussi la situation a changé. La totalité des fonctionnalités des devices ne sont pas couvertes par des plugins Cordova et certains de ces plugins ne sont pas disponibles sur toutes les plateformes (généralement ils le sont au moins sur iOS et Android), mais avec plus de <a href="https://cordova.apache.org/plugins/" target="_blank">1000 plugins</a> enregistrés, il est assez rare d'avoir à développer le sien (sauf dans des cas métiers ou techniques spécifiques).

Niveau outillage, plusieurs acteurs ont investi sur ce Framework. On retrouve plusieurs CLI différentes (<a href="https://github.com/Microsoft/TACO" target="_blank">taco</a>, <a href="http://ionicframework.com/docs/cli/" target="_blank">ionic</a>) proposant des fonctionnalités packagées par rapport à la CLI de Cordova, des Frameworks UI (<a href="http://ionicframework.com/" target="_blank">ionic</a>), des plugins et même une <a href="https://www.visualstudio.com/fr-fr/features/cordova-vs.aspx" target="_blank">intégration avancée dans Visual Studio</a> (templates de projet, compilation et debug sur Windows, Android et iOS, etc.).

De manière générale, on constate clairement une évolution ces dernières années dans la qualité des applications Cordova que l'on est capable de délivrer. Alors qu'on avait un niveau de finition moyen il y a encore à peine 2 ans (majoritairement à cause des navigateurs mobiles qui n'étaient pas assez performants et l'absence d'animations CSS3), on est capable d'offrir aujourd'hui des applications fluides avec une ergonomie poussée, de niveau équivalent à une application native.

## Comment s'y prendre pour réussir une application Cordova / Web ?

Il n'est pas évident de réussir une application Cordova, et pour mettre un maximum de chance de son côté, il convient de faire attention à plusieurs points.

### Application Web soignée

Le prérequis indispensable à la réussite d'une application Cordova est d'avoir une application Web de qualité. Le rôle de Cordova étant _simplement_ d'encapsuler une application Web, la qualité de l'application hybride sera directement liée à la qualité de l'application Web.

Le premier point important est d'avoir une application qui s'exécute côté client (c’est-à-dire une SPA, par exemple via les Framework AngularJS, React, etc.). Si ce n'est pas le cas, Cordova ne pourra pas embarquer l'application Web en local et sera obligé d'effectuer des requêtes HTTP pour récupérer chaque page, ce qui fournira une expérience d'utilisation pas aboutie (il existe cependant des techniques pour aspirer un site Web afin de générer des pages statiques).

Le niveau de finition de l'application est également fondamental. Il faut que l'interface soit fluide, que l'utilisateur ait des retours visuels, qu'il ait des indicateurs de chargements. User et abuser d'animations et de transitions rendra l'application plus agréable. Il faut également prendre soin de _gommer_ les traces de l'application Web, c’est-à-dire supprimer les `outline`, enlever la possibilité de sélectionner n'importe quel texte, etc. Enfin, il faut travailler la partie Responsive (voir Adaptive en fonction des applications) pour adapter l'application à toutes les tailles de device.

Dans certains cas, proposer une expérience d'utilisation offline, dégradée ou totale, peut être un vrai plus. Le stockage de données peut être fait en local (via `localStorage`, `IndexedDB` ou directement sur le device) ce qui permettra à l'application d'afficher ces données même sans réseau, et de pouvoir enregistrer les actions de l'utilisateur afin de les synchroniser une fois une connexion retrouvée.

### Outillage

S'outiller correctement est également décisif dans la réussite d'une application. Il y a quelques années, on avait un écosystème Cordova assez pauvre. Aujourd'hui, on a beaucoup d'outils nous permettant de gagner en productivité et en qualité.

Le premier outil est évidemment Visual Studio. Même pour les gens ayant horreur de Microsoft, il est difficile de trouver un IDE proposant une expérience de développement aussi aboutie sur Cordova. Templates de projets, éditeur graphique pour le fichier config.xml et les plugins, compilation dans l'IDE, debug cross-platform, etc. tout y est pour commencer efficacement un développement Cordova. Si vous n'êtes pas sur Windows ou que vous êtes allergique à Visual Studio, de nombreux autres IDE proposent également une intégration de Cordova intéressante.

L'IDE, bien que important, n'est pas le seul outil. Le moteur de build est également à ne pas sous-estimer. De nombreux services de build proposent maintenant la possibilité d'exécuter des modules npm, et donc par extension la CLI de Cordova. C'est par exemple le cas de TFS (ou VSTS) qui fournit des activités permettant de compiler une application Cordova afin de générer les applications natives. On peut même aller plus loin en utilisant des services comme HockeyApp pour déployer des versions de tests de l'application sur des devices à chaque build.

![Build VSTS](https://sebastienollivier.blob.core.windows.net/blog/1code=4applications/build-vsts.png =903x551 "Build VSTS")

![Release VSTS](https://sebastienollivier.blob.core.windows.net/blog/1code=4applications/release-vsts.png =1376x286 "Release VSTS")

Les copies d'écrans précédentes illustrent un exemple de build VSTS compilant l'application Cordova pour Windows et Android et d'une release VSTS déployant l'application Android via HockeyApp.

### Un seul code de base

Assez souvent, lorsque l'on crée une application Cordova, on en profite également pour créer une application Web en partageant le même code. Bien que l'application soit identique, les environnements d'hébergement Web et Cordova sont totalement différents (ASP.NET Core par exemple versus applications natives) ce qui implique inévitablement des différences de code (le téléchargement de fichiers par exemple se gère de manière différente en Web et en applications natives).

La tentation peut être forte de dupliquer le code afin de gérer les spécificités de chaque type d'hébergement. Le problème est que cette duplication introduit nécessairement des problématiques de synchronisation (corriger un bug doit être fait plusieurs fois, l'implémentation d'une feature doit être répercutée sur plusieurs branches, etc.). On perd partiellement la productivité qu'on avait gagnée avec Cordova et, plus gênant, on se risque à introduire des bugs / régressions ou des décalages. Afin d'éviter ça, l'idée est de considérer le Web comme une plateforme (au même titre que Android, Windows et iOS) et de créer un système de build permettant de générer l'application dans un conteneur Web (ASP.NET Core par exemple).

Je ne vais pas m'étendre plus sur ce point, qui fera l'objet d'un autre article, mais l'une des solutions est de partager tout le code au même endroit puis de créer un ensemble de tâches Gulp qui seront chargées de copier l'application dans le bon conteneur (projet Cordova ou projet ASP.NET Core pour le Web par exemple) en fonction de la plateforme cible.

### Compétences Web Front

Un autre point à ne pas négliger concerne les compétences requises pour créer une application Cordova. L'utilisation du triptyque HTML / CSS et JavaScript permet de réutiliser ses connaissances Web. Mais si l'équipe ne possède pas quelques bases dans le développement Front, notamment pour tout ce qui concerne l'optimisation (`translate3d` pour forcer l'accélération matérielle des animations, `requestAnimationFrame` pour demander un nouveau rendu de la page qu'aux moments clefs, etc.), le développement de l'application peut ne pas être aussi fluide et facile que ce que l'on avait prévu.

Le Framework Cordova en lui-même ne pose pas problème puisqu'il est plutôt simple d'utilisation.

### Tests sur devices
Le dernier point que je voulais évoquer concerne les tests sur devices. Lorsque l'on développe une application Cordova, il est toujours plus facile de tester directement depuis le navigateur (on profite de tous les outils permettant d'améliorer notre productivité, comme <a href="https://www.browsersync.io/" target="_blank">browserSync</a>). Mais le résultat sur navigateur n'est pas toujours fidèle à ce qu'on aura sur device (que ce soit niveau UI avec quelques différences dans l'interprétation du CSS ou niveau fluidité avec des performances totalement différentes).

Il est donc indispensable de tester tout au long du développement son application sur devices, afin d'éviter des surprises en fin de projet. En fonction des cibles, essayez de vous équiper d'une flotte en conséquence pour avoir des tests significatifs.

## Alors je fonce ?
Evidemment, Cordova n’est pas la solution ultime à utiliser pour chaque développement d'applications. Le développement natif est toujours d'actualité dans bien des cas (performances cruciales, besoin d'interactions poussées avec l'OS, etc.) et d'autres Framework cross-platform, comme Xamarin, propose des solutions toutes aussi efficaces.

L'intérêt majeur de Cordova est de pouvoir factoriser la totalité du code de l'application afin de cibler toutes les plateformes. En réalité, il est quasi impossible de partager 100% du code, chaque plateforme ayant ses spécificités et n'implémentant pas de la même manière les standards W3C. Mais on se retrouve généralement à un partage de l'ordre de 90% de l'application (voici un exemple de différence d'implémentation entre plateformes : <a href="http://blog.infinitesquare.com/b/kalbrecht/archives/implementer-une-googlemap-dans-une-application-cordova-avec-angularjs-et-typescript" target="_blank">http://blog.infinitesquare.com/b/kalbrecht/archives/implementer-une-googlemap-dans-une-application-cordova-avec-angularjs-et-typescript</a>).<br>Si vous avez besoin de créer un site web, l'intérêt est encore plus élevé puisque le code de vos applications pourra également être partagé avec le site web. Dernier point, si vous avez l'habitude du développement Web (plutôt orienté Front), Cordova est également une bonne solution puisque vous pourrez capitaliser sur vos compétences actuelles.

Par contre, si vous êtes plus à l'aise avec du C# / Xaml ou que vous souhaitez absolument avoir une UI adaptée aux guidelines de chaque plateforme (sachant que la tendance est aujourd'hui de développer sa propre identité graphique plutôt que de s'adapter aux plateformes), la solution Xamarin semble plus adaptée.

Bon développement cross-platform.

