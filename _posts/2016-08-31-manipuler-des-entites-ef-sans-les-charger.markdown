---
layout: post
title: 'Manipuler des entités EF sans les charger'
date: 2016-08-31
categories: net
resume: 'Souvent décrié à cause de problèmes de performances, Entity Framework peut se réveler très véloce si l''on sait correctement l''utiliser. Voyons comment manipuler des entités sans les charger.'
tags: perf performances ef entity-framework
---
Lorsque l'on utilise Entity Framework sans conna&icirc;tre son fonctionnement, on peut très vite se retrouver dans la situation où l'on génère du SQL non performant. L'une des erreurs que l'on rencontre le plus souvent consiste à récupérer trop d'entités. On va voir dans cet article comment on peut optimiser nos traitements SQL en évitant de charger inutilement ces entités.

## Mettre à jour une entité

Le premier cas que je rencontre concerne la mise à jour d'une entité. Le réflexe est d'avoir le code suivant :

```csharp
var post = await context.Posts.FirstOrDefaultAsync(p => p.Id == id);

if(post == null) 
{
    throw new NotFoundException();
}

post.Title = newTitle;
post.Content = newContent;
post.Url = newUrl;

await context.SaveChangesAsync();
```

Ce code va déclencher les requêtes SQL suivantes :

```sql
SELECT [...] FROM Posts Where Id = @id

UPDATE Posts
SET Title = @title, Content = @content, Url = @url
WHERE Id = @Id
```
La première requête est inutile. On récupère des données qui ne seront pas utilisés pour l'UPDATE.

Pour optimiser le code SQL généré, l'idée est de jouer avec le contexte d'Entity Framework pour lui faire croire que l'entité a été chargée puis d'effectuer les modifications.

```csharp
var post = new Post { Id = id };

context.Entry(post).State = EntityState.Unchanged;

post.Title = newTitle;
post.Content = newContent;
post.Url = newUrl;

await context.SaveChangesAsync();
```
L'instruction `context.Entry` permet d'attacher l'entité Post dans le contexte EF. De cette façon, Entity Framework pensera que l'entité a été chargée et il trackera tous les changements effectués. Le code précédent génèrera uniquement une requête SQL d'Update.

Pour que ce code fonctionne en toute circonstance, il est nécessaire de vérifier que l'entité n'est déjà attachée au contexte. En effet, si on essaye d'attacher l'entité alors qu'elle existe déjà, Entity Framework lancera une exception de type InvalidOperationException avec un message du type _Attaching an entity of type '***' failed because another entity of the same type already has the same primary key value_. Pour solidifier le code précédent, il faudrait rajouter la vérification suivante :

```csharp
var post = context.Set<Post>().Local.FirstOrDefault(p => p.Id == id);
if (post == null)
{
    post = new Post { Id = id };
    context.Entry(post).State = EntityState.Unchanged;
}
```

## Mettre à jour des relations

Dans la même idée, je vois souvent du code qui ressemble à :

```csharp
var post = await context.Posts.FirstOrDefaultAsync(p => p.Id == postId);
var category = await context.Categories.FirstOrDefaultAsync(c => c.Id == categoryId);

if(post == null || category == null) 
{
    throw new NotFoundException();
}

post.Categories.Add(category);

await context.SaveChangesAsync();
```

Pour lier une catégorie à un article, on va récupérer l'article, puis la catégorie pour enfin ajouter la relation. Ce code génèrera les requêtes SQL suivantes :

```sql
SELECT [...] FROM Posts Where Id = @postId

SELECT [...] FROM Categories Where Id = @categoryId

INSERT INTO PostCategories(PostId, CategoryId) VALUES (@postId, @categoryId)
```

Comme pour l'exemple précédent, on pourrait se passer des deux premières requêtes. Voici le code optimisé :

```csharp
var post = new Post { Id = postId };
var category = new Category { Id = categoryId };

context.Entry(post).State = EntityState.Unchanged;
context.Entry(category).State = EntityState.Unchanged;

post.Categories.Add(category);

await context.SaveChangesAsync();
```

Là encore, on utilise la méthode `context.Entry` pour attacher au contexte nos deux entités. Ensuite, on ajoute simplement la relation et le SQL généré ne contient plus que la requête INSERT.

En attachant manuellement des entités sur le contexte, on peut assez facilement optimiser le code SQL généré.

Bon requêtage !

