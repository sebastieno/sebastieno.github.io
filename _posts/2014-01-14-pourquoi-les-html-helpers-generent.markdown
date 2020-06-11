---
layout: post
title: 'Pourquoi les HTML Helpers génèrent une mauvaise valeur ?'
date: 2014-01-14
categories: asp-net-mvc
resume: 'Pourquoi est-ce que les HTML Helpers génèrent une mauvaise valeur ? Pourquoi est-ce qu''ils utilisent l''ancienne valeur alors que je l''ai modifié dans le controller ? Voyons ça ensemble.'
tags: html-helpers
---
## Symptôme

Dans une action HttpPost ASP.NET MVC, je modifie le modèle posté et je le renvoie à la vue. La vue générée ne prend pas en compte les modifications effectuées lorsque j'utilise les HTML Helpers !

# Exemple :

La vue est la suivante (les champs Login et Nom sont obligatoires, et le login doit faire 10 caractères maximum):

![Exemple de formulaire](https://sebastienollivier.blob.core.windows.net/blog/pourquoi-les-html-helpers-generent-une-mauvaise-valeur/generate-wrong-values-filled-form.png =250x167 "Exemple de formulaire")

L'action du controller est la suivante (on tronque simplement la propriété `Login` pour la limiter à 10 caractères) :

```csharp
[HttpPost]
public ActionResult Create(SampleModel model)
{
    if (ModelState.IsValid)
    {
        // Do business stuff
        return RedirectToAction("Index", "Home");
    }

    if (!string.IsNullOrEmpty(model.Login))
    {
        model.Login = model.Login.Substring(0, 10);
    }

    return View(model);
}
```

La vue retournée en cas de non validité du modèle est la suivante (le tronquage de la propriété `Login` n'a pas été prise en compte) :

![Exemple de formulaire en erreur](https://sebastienollivier.blob.core.windows.net/blog/pourquoi-les-html-helpers-generent-une-mauvaise-valeur/generate-wrong-values-form-with-errors.png =500x161 "Exemple de formulaire en erreur")

## Pourquoi ?

Lorsque l'on poste un formulaire HTML, ASP.NET MVC va reconstruire le modèle (via le ModelBinder) et va stocker son état dans le ModelState, c'est à dire s'il est valide ou non (en fonction des DataAnnotations notamment).

Au moment où l'on va regénérer la vue, les HTML Helpers vont se baser sur le ModelState pour rendre les contrôles, en prenant en compte l'état de chaque propriété. La modification d'une propriété du modèle dans l'action ne va pas mettre à jour le ModelState et ne sera donc pas prise en compte par les HTML Helpers lors de la génération de la vue.

## Solutions

ASP.NET MVC propose trois solutions pour réappliquer des valeurs sur le modèle.

### ModelState.Clear

La première solution consiste à appeler la méthode `Clear` du ModelState juste avant l'envoi du modèle à la vue. Cette méthode va supprimer toutes les entrées du ModelState (valeurs, états, messages d'erreurs, etc.) de toutes les propriétés du modèle. Les HTML Helper vont alors se baser sur les valeurs du modèle au lieu des valeurs du ModelState.

```csharp
[HttpPost]
public ActionResult Create(SampleModel model)
{
    [...]

    ModelState.Clear();
    return View(model);
}
```

![Formulaire en erreur, après Clear()](https://sebastienollivier.blob.core.windows.net/blog/pourquoi-les-html-helpers-generent-une-mauvaise-valeur/generate-wrong-values-form-after-modelstateclear.png =250x156 "Formulaire en erreur, après Clear()")

### ModelState.Remove

Si on veut agir plus finement, on peut appeler la méthode `Remove` du ModelState en passant le nom d'une propriété du modèle. Les entrées du ModelState associées à cette propriété seront supprimées (valeur, état, messages d'erreurs, etc.) et les nouvelles valeurs seront appliquées.

```csharp
[HttpPost]
public ActionResult Create(SampleModel model)
{
    [...]

    ModelState.Remove("Login");
    return View(model);
}
```

![Formulaire en erreur après Remove()](https://sebastienollivier.blob.core.windows.net/blog/pourquoi-les-html-helpers-generent-une-mauvaise-valeur/generate-wrong-values-form-after-modelstateremove.png =400x182 "Formulaire en erreur après Remove()")

### Ne pas utiliser les HTML Helpers

La dernière solution consiste à ne pas utiliser les HTML Helpers pour générer la vue. De cette manière, on ne prendra pas en compte le ModelState et les valeurs modifiées seront prises en compte dans la vue. Attention cependant, ne pas utiliser les HTML Helpers nécessite de générer manuellement tous les validateurs (en fonction des _DataAnnotations_).


Voilà !


