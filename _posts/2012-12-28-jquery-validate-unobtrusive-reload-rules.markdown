---
layout: post
title: 'Rafraîchir les règles de validation avec Unobtrusive Validation'
date: 2012-12-28
categories: javascript
resume: 'Voici un helper permettant de forcer le rechargement des règles de validation déclarées en utilisant jQuery Unobtrusive Validation.'
tags: jquery jquery-validate jquery-unobtrusive-validation
---
Dans la série post concernant Javascript, je vais vous parler aujourd'hui de <a href="http://docs.jquery.com/Plugins/Validation" target="_blank">jQuery Validation</a> et plus particulièrement de <a href="http://nuget.org/packages/Microsoft.jQuery.Unobtrusive.Validation" target="_blank">Unobtrusive Validation</a>.

<a href="http://docs.jquery.com/Plugins/Validation" target="_blank">jQuery Validation</a> est un plugin jQuery permettant de gérer la validation d'un formulaire côté client. Pour chaque `input` (ou `select` ou `textarea`) du formulaire, le plugin va stocker dans un objet JS toutes les règles de validation et les vérifier au submit. Si le formulaire n'est pas valide, le submit est annulé.


<a href="http://nuget.org/packages/Microsoft.jQuery.Unobtrusive.Validation" target="_blank">Unobtrusive Validation</a> est utilisé pour charger toutes les règles de validation d'un formulaire de manière non intrusive (unobtrusive), c'est à dire sans avoir de JS dans la page HTML. Ce plugin se sert des attributs `data-val-*` pour déduire les règles de validation au chargement de la page (évènement `document read` de jQuery).

Lors d'un projet où l'on modifiait dynamiquement les règles de validation en fonction des choix de l'utilisateur (ajout de nouvelles zones du formulaire, ajout / suppression de champs requis, etc.), on s'est rendu compte qu'une fois les règles de validation chargées, il n'était pas possible de demander le rechargement de ces règles.


Voici deux solutions pour forcer ce rechargement.

## Solution brutale

La solution brutale consiste à supprimer le validateur associé au formulaire puis de le reconstruire en utilisant la syntaxe suivante :

```js
$('#formid').removeData('validator')
$('#formid').validate()
```

La première ligne va supprimer l'objet JS contenant les données de validation. La deuxième ligne va appeler le plugin jQuery Validation qui va recréer les règles.
Cette solution présente l'avantage d'être rapide à mettre en œuvre. Le désavantage se trouve au niveau des performances puisqu'on va reparser l'ensemble du formulaire au lieu de la zone dynamiquement modifiée. Cette méthode n'est donc pas satisfaisante pour des gros formulaires.

## Parse Dynamic Fields

La deuxième solution est beaucoup plus élégante puisqu'il s'agit de parser uniquement une zone du formulaire pour recréer les règles de validation.

Quelqu'un a posté une extension JS permettant de rajouter les règles de validation d'une nouvelle zone du formulaire (c'est à dire dont les règles n'ont pas encore été chargées) : <a href="http://xhalent.wordpress.com/2011/01/24/applying-unobtrusive-validation-to-dynamic-content/" target="_blank">http://xhalent.wordpress.com/2011/01/24/applying-unobtrusive-validation-to-dynamic-content/</a>. Cette extension est un excellent début puisqu'elle permet d'ajouter des règles de validation via le principe d'unobstrusive sans avoir à recréer tout le validateur (et donc reparser tout le formulaire).

La limitation de ce script est qu'il ne permet que de rajouter des règles de validation contenues dans une nouvelle zone du formulaire. J'ai donc pris le temps d'enrichir ce script pour qu'il rafraichisse les règles d'une zone d'un formulaire, que cette zone soit nouvelle ou pas.


Pour demander le rafraichissement d'une zone d'un formulaire, il suffit d'utiliser la syntaxe suivante, où `formAreaId` correspond à l'id de la zone du formulaire à rafraîchir :

```js
$.validator.unobtrusive.reloadRules("#formAreaId");
```

Ce script prend en charge :
* L'ajout de règles à partir d'une nouvelle zone de formulaire
* La mise à jour des règles pour une zone existante d'un formulaire (ajout, modification et suppression des règles).

Comme d'habitude, j'ai créé un package nuget pour faciliter le déploiement dans vos solutions Visual Studio : <a href="https://nuget.org/packages/UnobtrusiveValidationReloadRules" target="_blank">_UnobtrusiveValidationReloadRules_</a>

Le fichier JS seul se trouve à l'adresse suivante : <a href="https://skydrive.live.com/redir?resid=2BAE7BB2DE0FBFC7!313" target="_blank">_sor.validate.unobtrusive.extend.js_</a>

A bientôt et bonnes fêtes.


