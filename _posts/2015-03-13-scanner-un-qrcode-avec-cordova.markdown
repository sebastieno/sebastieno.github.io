---
layout: post
title: 'Scanner un QR Code avec Cordova'
date: 2015-03-13
categories: cordova
resume: 'Scannons des QR Code depuis une application Cordova en tirant parti de l''utilisation d''un plugin (permettant  d''accéder aux fonctionnalités natives du device).'
tags: qrcode cordova plugins barcodescanner
---
Lorsque l'on crée une application Cordova, en encapsulant une application Web, il est souvent nécessaire d'accéder aux fonctionnalités natives du device. On va voir dans cet article comment on peut interagir avec le device pour scanner des QR Codes.

## Pourquoi ne pas utiliser la Stream API ?

Il est possible d'accéder au stream vidéo d'une webcam en JavaScript en utilisant la <a href="http://www.w3.org/TR/mediacapture-streams/" target="_blank">Stream API</a>. Cette API est encore à l'état de Working Draft, c’est-à-dire non définitive et amenée très probablement à évoluer, et est encore peu supportée par les navigateurs : 

![Can I Use StreamAPI](https://sebastienollivier.blob.core.windows.net/blog/qrcode-cordova/caniuse-streamapi.png =1221x474 "Can I Use StreamAPI")
_Source : <a href="http://caniuse.com/#feat=streami" target="_blank">http://caniuse.com/#feat=stream</a>_

Cette solution n'est donc pas viable si l'on souhaite que cela fonctionne sur des devices iOS, Android et Windows Phone. La seule possibilité est donc de passer par un plugin Cordova pour accéder aux fonctionnalités natives du device.

## Comment fonctionne un plugin Cordova ?  

Un plugin Cordova permet de créer une communication entre l'application web, qui est encapsulée par Cordova, et du code natif de la plateforme cible.

Un plugin Cordova est composé d'un contrat JavaScript, définissant les fonctions qui seront exposées à l'application web, et d'une implémentation par plateforme (généralement iOS, Android et Windows Phone). A la génération d'une application, Cordova injectera l'implémentation correspond à la plateforme cible, ce qui permet d'avoir un même code JavaScript qui fonctionnera sur toutes les plateformes (en tout cas celles supportées par le plugin).

![Cordova plugin mecanism](https://sebastienollivier.blob.core.windows.net/blog/qrcode-cordova/cordovaplugin-mecanism.png =392x598 "Cordova plugin mecanism")

## Utilisation du plugin de lecteur de QR Code

Depuis Visual Studio 2015, l'installation d'un plugin est plutôt facile. Il suffit d'ouvrir le fichier `config.xml` du projet Cordova, d'aller dans l'onglet Plugins et de choisir le plugin à installer. Si votre plugin n'est pas listé dans les plugins Core, vous pouvez renseigner son adresse (local ou un repository Git) depuis l'onglet Custom :
  
![Plugin install in VS 2015](https://sebastienollivier.blob.core.windows.net/blog/qrcode-cordova/vs2015-plugininstall.png =1143x416 "Plugin install in VS 2015")

L'installation depuis la CLI n'est pas beaucoup plus dure, il suffit d'utiliser la commande _cordova plugin add_ en spécifiant le nom du plugin ou son adresse :

```javascript
cordova plugin add https://github.com/sebastieno/phonegap-plugin-barcodescanner
```

_Note: Le plugin de scan de QR Code présente des bugs au niveau de l'implémentation Windows Phone. En attendant l'acceptation d'une Pull Request contenant les correctifs, vous pouvez trouverez une version fonctionnelle à l'adresse suivante : <a href="https://github.com/sebastieno/phonegap-plugin-barcodescanner" target="_blank">https://github.com/sebastieno/phonegap-plugin-barcodescanner</a>_.

Une fois installé, le plugin sera présent dans le dossier _plugins_ de votre projet :

![VS 2015 Solution](https://sebastienollivier.blob.core.windows.net/blog/qrcode-cordova/vs2015-solution.png =302x438 "VS 2015 Solution")

L'utilisation du plugin se fait alors en appelant les fonctions définies par le contrat JavaScript. Pour notre plugin, il s'agit de la fonction _cordova.plugins.barcodeScanner.scan_ (vous trouverez plus d'informations directement sur le GitHub <a href="https://github.com/phonegap/phonegap-plugin-barcodescanner" target="_blank">https://github.com/phonegap/phonegap-plugin-barcodescanner</a>):

```js
cordova.plugins.barcodeScanner.scan(function (result) {
	// result contient le résultat du scan
}, function (err) {
	// err contient la cause de l'erreur
});
```

Ce code va appeler le code natif du plugin, qui prendra la main pour scanner un QR Code puis renverra le résultat du scan à l'application Web via un callback. Aussi simple que ça !

Bons scans !
