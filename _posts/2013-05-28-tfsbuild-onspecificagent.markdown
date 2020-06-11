---
layout: post
title: 'Déclencher une build TFS sur un agent particulier'
date: 2013-05-28
categories: tfs
resume: 'Quelles sont les deux méthodes permettant de spécifier sur quel agent doit être exécutée une build ?'
tags: tfs build vsts agent
---
Lorsque l'on utilise les builds TFS sur un projet, il peut être intéressant de spécifier sur quel agent doit être exécutée une build spécifique.

TFS nous propose plusieurs solutions pour répondre à ce besoin.

## Spécifier explicitement l'agent de build pour une définition de build

Quand on crée une définition de build, on peut spécifier quel agent doit exécuter la build. Sur l'écran de définition d'une build, cela se passe dans la zone _Agent Settings_ de la section _Advanced_. Le champ _Name Filter_ permet de renseigner le nom de l'agent qui devra exécuter la build, via une liste déroulante (par défaut la valeur _*_ indiquant n'importe quel agent disponible).

![Agent settings d'une définition de build](https://farm4.staticflickr.com/3732/8800732995_b4344a65ae_o.png =703x137 "Agent settings d'une définition de build")

De cette manière, la build sera systématiquement exécutée sur l'agent renseigné.

## Utiliser les Tags pour déterminer quel agent utiliser

La deuxième solution consiste à utiliser les Tags. On peut tagguer les agents et les définitions de builds avec des mots clefs, ces tags permettant au controller TFS de choisir quel est l'agent qui doit exécuter une build.

Pour tagguer un agent, il faut aller dans la vue _Team Explorer_ et cliquer sur _Manage Build Controllers..._.

![Manage Build Controllers...](https://farm3.staticflickr.com/2879/8811318872_02f0cca391_o.png =405x260 "Manage Build Controllers...")

Sélectionnez un agent de build puis cliquez sur _Properties_. A partir de là, la zone _Tags_ vous permettra d'ajouter des mots clefs.

![Tags d'un agent](https://farm4.staticflickr.com/3789/8870672373_915d0a2909_o.png =531x445 "Tags d'un agent")

Pour définir les tags requis pour exécuter une build, sur l'écran de définition d'une build il faut aller dans la zone _Agent Settings_ de la section _Advanced_. Le champ _Tags Filter_ permet de définir le ou les tags de la build. Le champ _Tag Comparison Operator_ permet de définir le type de comparaison entre les tags de la build et ceux de l'agent (_MatchExactly_ ou _MatchAtLeast_).

![Tags d'une définition de build](https://farm4.staticflickr.com/3773/8871282048_d0ebf19511_o.png =728x122 "Tags d'une définition de build")

De cette manière, à l'exécution de la build, le controller va sélectionner l'agent qui devra exécuter cette build en fonction des tags renseignés et du type de comparaison.

Bonnes builds !

