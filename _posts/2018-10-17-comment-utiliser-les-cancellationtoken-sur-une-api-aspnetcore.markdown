---
layout: post
title: 'Comment utiliser les CancellationToken sur une API ASPNET Core ?'
date: 2018-10-17
categories: asp-net-mvc
resume: 'L''utilisation de CancelationToken sur une API ASPNET Core est extrêmement simple à mettre en place, et pourtant personne ne le fait. Découvrons ce qu''il en est.'
tags: cancellationtoken aspnetcore
---
Les <a href="" title="CancellationToken" target="_blank">CancellationToken</a> sont des objets permettant d'annuler des tâches asynchrones. Dans le cas d'une API ASPNET Core, elles peuvent être utilisées Core afin de gérer l'annulation des requêtes HTTP par le client.

Et pour l'utiliser, il faut... simplement déclarer un paramètre de type `CancellationToken` sur vos actions (et bien sûr penser à passer le paramètre aux méthodes sous-jacentes) :

```csharp
[HttpGet("")]
public async Task<IActionResult> Login([FromBody]LoginRequest loginRequest, CancellationToken cancellationToken)
{
    return await this.authenticationManager.Login(loginRequest.Login, loginRequest.Password, cancellationToken);
}
```

ASPNET Core va automatiquement binder tous les paramètres de type `CancellationToken` de vos actions à la propriété `HttpContext.RequestAborted`, via le model binder <a href="https://github.com/aspnet/AspNetCore/blob/master/src/Mvc/src/Microsoft.AspNetCore.Mvc.Core/ModelBinding/Binders/CancellationTokenModelBinder.cs" title="CancellationTokenModelBinder" target="_blank">CancellationTokenModelBinder</a>

Et voilà, rien de plus simple à mettre en place et pourtant très intéressant.

Bons CancellationToken !

