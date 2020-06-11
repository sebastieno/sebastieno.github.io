---
layout: post
title: 'Définir le CacheControl de tous les blobs d''un container Azure'
date: 2015-05-13
categories: azure
resume: 'Voici un script Powershell permettant de définir le header ''CacheControl'' à l''ensemble des blobs d''un container Azure.'
tags: azure blob blob-storage cache-control headers
---
L'entête HTTP _CacheControl_ permet de spécifier la durée pendant laquelle la ressource liée doit être mise en cache (pour plus d'informations sur cet entête <a href="http://fr.wikipedia.org/wiki/Cache-Control" target="_blank">http://fr.wikipedia.org/wiki/Cache-Control</a>).

Sur mon blog perso (<a href="http://sebastienollivier.fr/blog" target="_blank">http://sebastienollivier.fr/blog</a>), les images des différents articles sont hébergés sous la forme de blobs Azure. Le header _CacheControl_ renvoyé par défaut est vide, ce qui veut dire qu'aucune stratégie de cache n'est définie, et donc que le client peut potentiellement aller récupérer plusieurs fois la même ressource dans un lapse de temps assez court. Pas très optimisé.

Dans le but d'améliorer le chargement des pages, j'ai voulu modifier le _CacheControl_ renvoyé par Azure sur l'ensemble des blobs du container utilisé par mon blog. Sauf qu'il n'y a pas de moyen rapide (càd via le portail Azure par exemple) de définir ce header pour l'ensemble des blobs (et autant dire que les faire un par un, ca peut prendre du temps). J'ai donc créé un script Powershell qui permet de faire ça.

```powershell
Add-Type -Path "c:\Program Files\Microsoft SDKs\Azure\.NET SDK\v2.5\bin\Microsoft.WindowsAzure.StorageClient.dll"
 
$accountName = "[set your account name]"
$accountKey = "[set your account key]"
$blobContainerName = "[set your container name]"
 
$storageCredentials = New-Object Microsoft.WindowsAzure.StorageCredentialsAccountAndKey -ArgumentList $accountName,$accountKey
$storageAccount = New-Object Microsoft.WindowsAzure.CloudStorageAccount -ArgumentList $storageCredentials,$true
$blobClient =  [Microsoft.WindowsAzure.StorageClient.CloudStorageAccountStorageClientExtensions]::CreateCloudBlobClient($storageAccount)

$cloudBlobContainer = $blobClient.GetContainerReference($blobContainerName)
$blobRequestOptions = new-object Microsoft.WindowsAzure.StorageClient.BlobRequestOptions;
$blobRequestOptions.UseFlatBlobListing = $true;
 
$cacheControlValue = "public, max-age=2592000" # 1 mois

$cloudBlobContainer.ListBlobs($blobRequestOptions) | foreach {  
    $_.Properties.CacheControl = $cacheControlValue
    $_.SetProperties()
}
```

Le script précédent se base sur la dll _Microsoft.WindowsAzure.StorageClient.dll_ (la version 2.5 dans mon cas) pour créer tous les objets nécessaires au parcours des blobs et à l'affectation du `CacheControl` (cache publique d'1 mois dans le code). J'aurais pu me baser sur le module Powershell d'Azure, mais je n'ai pas réussi à trouver les commandes nécessaires.

Pour utiliser ce script dans votre contexte, il suffit de renseigner les variables `$accoutName`, `$accountKey` et `$blobContainerName`. Vous pouvez également modifier la variable `$cacheControlValue` pour définir la valeur à renseigner dans le header `CacheControl`.

Vous pouvez trouver ce script <a href="http://1drv.ms/1HgstcO" target="_blank">ici</a>. Bonne mise en cache !
