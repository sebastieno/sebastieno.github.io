---
layout: post
title: 'Authentification via Windows Azure Active Directory'
date: 2013-04-10
categories: azure
resume: 'Quelles sont les modifications apportées par le passage de WAAD (Windows Azure Active Directory) de la version Preview à la version de Production ?'
tags: waad aad authentication
---
Mon dernier post (<a href="https://sebastienollivier.fr/blog/azure/authentification-via-windows-azure-active-directory/">https://sebastienollivier.fr/blog/azure/authentification-via-windows-azure-active-directory/</a>) expliquait comment déléguer l’authentification d’une application à Windows Azure Directory, en version Preview. WAAD vient de passer en version Production ce qui amène quelques améliorations.

Les informations du dernier post sont toujours d’actualité mais le portail a évolué et nous permet d’administrer notre tenant WAAD sans avoir à passer par des commandes Powershell.

Je vais reprendre la première partie du post précédent, qui permettait d’enregistrer une application sur WAAD. J’expliquerai comment profiter des nouvelles features du portail Azure pour ne pas avoir à utiliser de Powershell 

## Enregistrement de l’application dans WAAD

Sur le portail Azure (<a href="https://manage.windowsazure.com/" target ="_blank">https://manage.windowsazure.com/</a>), vous trouverez un onglet _Active Directory_ qui permet d’administrer vos tenants WAAD et vos comptes ACS.

![Onglet Active Directory du portail Azure](https://farm9.staticflickr.com/8265/8637613400_77aa4dd8bb_o_d.png =222x172 "Onglet Active Directory du portail Azure")

Le lien _Directory_ permet d’afficher la liste de vos tenants.

![Liste des tenants WAAD](https://farm9.staticflickr.com/8109/8636506271_3f0097eee6_o.png =723x174 "Liste des tenants WAAD")

Au clic sur un tenant, vous naviguerez vers son interface d’administration. A partir de là, le lien _Integrated Apps_ vous permettra de définir les applications enregistrées sur ce tenant.

![Applications enregistrées sur WAAD](https://farm9.staticflickr.com/8543/8637613408_5bbf591dfb_o.png =713x166 "Applications enregistrées sur WAAD")
Pour ajouter une nouvelle application, il suffit de cliquer sur le lien _Add_. La première étape du wizard va demander de renseigner un nom d’application ainsi qu’un type d’accès. Si l’application n’a pas besoin d’accéder aux Graph API, le type d’accès _Single Sign-on_ suffira.

![Wizard d'ajout d'une application dans WAAD](https://farm9.staticflickr.com/8541/8637613406_5071d97811_o.png =390x297 "Wizard d'ajout d'une application dans WAAD")

La deuxième étape du wizard va demander de renseigner une _App URL_, correspondant à l’URL de l’application et un _App ID URI_, correspondant à un identifiant unique d’application.

![Wizard d'ajout d'une application dans WAAD](https://farm9.staticflickr.com/8406/8638027422_cf51316b24_o.png =327x230 "Wizard d'ajout d'une application dans WAAD")

Après validation, l'application sera déclarée dans WAAD et pourra lui déléguer son authentification

## Délégation de l’authentification à WAAD

Pour configurer l’application, vous pouvez vous reporter au post précédent :<br /><a href="https://sebastienollivier.fr/blog/azure/authentification-via-windows-azure-active-directory/">https://sebastienollivier.fr/blog/azure/authentification-via-windows-azure-active-directory/</a>

Les seuls changements concernent l'URL du fichier de metadata et l’Application Uri à utiliser via le Wizard de configuration. Vous pouvez trouver les nouvelles valeurs directement sur le portail Azure :

![Informations de l'application](https://farm9.staticflickr.com/8262/8636506251_910b3a219d_o.png =716x517 "Informations de l'application")

N'oubliez pas de rajouter l'attribut `reply` au noeud `wsFederation` et le tour est joué.

Beaucoup plus simple qu'avec Powershell :) !


