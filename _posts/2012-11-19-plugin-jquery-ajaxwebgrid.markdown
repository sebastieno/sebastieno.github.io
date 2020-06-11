---
layout: post
title: 'Plugin jQuery AjaxWebGrid'
date: 2012-11-19
categories: javascript
resume: 'Retrouvez ici un plugin jQuery très light permettant de transformer une grille HTML en grille Ajax supportant notamment le tri, la pagination et les filtres.'
tags: ajaxwebgrid
---
**[Update : Une nouvelle version du plugin jQuery AjaxWebGrid est disponible. Pour plus d'informations : <a href="http://sebastienollivier.fr/blog/javascript/plugin-jquery-ajaxwebgrid-1-1-0/" target="_blank">http://sebastienollivier.fr/blog/javascript/plugin-jquery-ajaxwebgrid-1-1-0/</a>]**

On trouve plein de plugins/composants/contrôles permettant de créer une grille (tableau HTML) avec paginations, tris et filtres en Ajax. Malheureusement, entre ceux qui sont payants, ceux qui sont lourds, ceux qui sont construits comme des usines à gaz ou ceux qui ne marchent que partiellement, il est parfois difficile de trouver le bon plugin/composant/contrôle.

Après pas mal de temps sans blogger, je fais ce post pour partager avec vous un plugin jQuery que j'ai créé et qui permet d'ajouter un comportement Ajax à une grille HTML. J’utilise ce plugin sur la plupart des projets ASP.NET MVC sur lesquels je travaille, pour créer rapidement une grille supportant le tri, la pagination, les filtres, le tout en Ajax.

## AjaxWebGrid

Pour l'utiliser, il faut commencer par rajouter une référence au fichier `sor.ajaxWebGrid.js`.

Il suffit ensuite d'utiliser la syntaxe jQuery suivante sur l'élément HTML représentant la grille :

```js
$("#usersGrid").asAjaxWebGrid({ gridActionUrl : '@Url.Action('List', 'User') })
```

Le fonctionnement est plutôt simple. Tous les liens contenus dans le `thead` (le tri par exemple) et dans le `tfoot` (la pagination par exemple) de la grille défini par le sélecteur jQuery (dans l'exemple précédent, le tableau HTML dont l'identifiant est `usersGrid`), seront transformés en liens Ajax.

Un appel Ajax sera effectué au clic d'un de ces liens, en utilisant l'url défini par le paramètre `gridActionUrl`. Le plugin utilisera les options `sortFieldNameProperty`, `sortDirectionProperty` et `pageIndexProperty` pour déterminer quels sont les paramètres à extraire de la query string du lien cliqué et les envoyer lors de la requête Ajax. Le HTML de la grille sera alors remplacé par le HTML retourné par l'appel Ajax (voir l'option `refreshGridSuccessCallback` pour modifier ce comportement).

Pour relancer manuellement le rafraîchissement du tableau (via une zone de filtre par exemple), utilisez la syntaxe suivante :

```js
$("#usersGrid").data("ajaxWebGrid").refreshGrid();
```

Les options du plugin sont les suivantes :
* `gridActionUrl` [Obligatoire] : URL de l'action MVC utilisée pour rafraîchir la grille.
*  `noResultContainerSelector` : Sélecteur jQuery correspondant à l'élément HTML lorsque la grille est vide. Si vous retournez un élément HTML différent de la grille  lorsqu'il n'y a pas de résultats à afficher, ce paramètre doit obligatoirement être renseigné.
* `sortFieldNameProperty` : Nom de la propriété de la query string utilisée pour spécifier le nom de la colonne à partir de laquelle les données sont triées. Par défaut, sa valeur est positionnée à "sortFieldName".
* `sortDirectionProperty` : Nom de la propriété de la query string utilisée pour spécifier l'ordre de trie (ascendant ou descendant). Par défaut, sa valeur est positionnée à "sortDirection".
* `pageIndexProperty` : Nom de la propriété de la query string utilisée pour spécifier la page courante affichée. Par défaut, sa valeur est positionnée à "page".
* `addCustomParameters` :  Fonction appelée avant chaque rafraîchissement Ajax. Elle permet d'ajouter des paramètres qui seront transmis au serveur lors de l'appel Ajax. Cette fonction ne prend pas de paramètres en entrée et doit renvoyer un dictionnaire de type string / string, où la clef correspond au nom du paramètre et la valeur correspond à la valeur du paramètre.
* `refreshGridSuccessCallback` : Fonction appelée après un rafraîchissement Ajax effectué avec succès. Par défaut, la fonction remplace l’élément HTML de la grille ou l’élément HTML généré lorsque la grille est vide (voir propriété `noResultContainerSelector`) par le HTML retourné par l’appel Ajax.
Les paramètres de cette fonction sont : le HTML retourné par le serveur, le sélecteur jQuery de la grille ainsi que les options de la grille.
* `refreshGridFailedCallback` : Fonction appelée après un rafraîchissement Ajax ayant provoqué une erreur. Par défaut, la fonction ne fait rien. Les paramètres de cette fonction sont : paramètres standard en cas d'erreurs Ajax (XMLHttpRequest, textStatus, errorThrown) ainsi que le sélecteur jQuery de la grille et les options de la grille.

## Utilisation avec ASP.NET MVC

Voici un exemple d'utilisation avec une WebGrid MVC simple :

### Controller ASPNET

```csharp
public ActionResult Index(int? page, string sortFieldName, string sortDirection)
{
    int totalRowNumber;

    UserManager manager = new UserManager();
    IEnumerable<User> users = manager.GetUsers([...]);

    UsersListModel model = new UsersListModel
    {
        Page = page.HasValue ? page.Value : 0,
        RowPerPage = RowPerPage,
        SortDirection = sortDirection,
        SortFieldName = sortFieldName,
        TotalRowNumber = totalRowNumber,
        Users = users
    };

    if (Request.IsAjaxRequest())
    {
        return PartialView(model);
    }

    return View(model);
}
```

### Vue Razor

```csharp
@{ 
    WebGrid grid = new WebGrid(sortFieldName: "sortFieldName",
        sortDirectionFieldName: "sortDirection",
        defaultSort: Model.SortFieldName,
        rowsPerPage: Model.RowPerPage);

    grid.Bind(Model.Users, autoSortAndPage: false, rowCount: Model.TotalRowNumber);

    if (Model.TotalRowNumber > 0)
    {
        grid.PageIndex = Model.Page > 0 ? Model.Page - 1 : 0;
    }

    grid.SortDirection = Model.SortDirection != "DESC" ?
        SortDirection.Descending : 
        SortDirection.Ascending;
}

@{
    List<WebGridColumn> gridColumns = new List<WebGridColumn>
    {
        new WebGridColumn
        {
            ColumnName = "Name",
            Header = "Nom",
            CanSort = true
        },
        new WebGridColumn
        {
            ColumnName = "FirstName",
            Header = "Prénom",
            CanSort = true
        },
        new WebGridColumn
        {
            ColumnName = "Age",
            Header = "Age",
            CanSort = true
        },
        new WebGridColumn
        {
            ColumnName = "Sex",
            Header = "Sexe",
            CanSort = true,
            Format = @<text>@if (item.Sex == Sex.M) 
                             { <text>Homme</text> } 
                             else { <text>Femme</text> }
                      </text>
        }
    };
}

@if (Model.TotalRowNumber > 0)
{
    @grid.GetHtml(htmlAttributes: new { id = "usersGrid" }, columns: gridColumns)
}
else
{
    <div id="noresults">
        <span>Aucun résultat.</span>
    </div>
}
```

### HTML Généré

```html
<table id="usersGrid">
    <thead>
        <tr>
            <th scope="col">
                <a href="/?sortFieldName=Name&amp;sortDirection=ASC">Nom</a>
            </th>
            <th scope="col">
                <a href="/?sortFieldName=FirstName&amp;sortDirection=ASC">Prénom</a>
            </th>
            <th scope="col">
                <a href="/?sortFieldName=Age&amp;sortDirection=ASC">Age</a>
            </th>
            <th scope="col">
                <a href="/?sortFieldName=Sex&amp;sortDirection=ASC">Sexe</a>
            </th>
        </tr>
    </thead>
    <tfoot>
        <tr>
            <td colspan="4">
                <a href="/?page=3">&lt;</a> 
                <a href="/?page=1">1</a> 
                <a href="/?page=2">2</a> 
                <a href="/?page=3">3</a> 
                4 
                <a href="/?page=5">5</a> 
                <a href="/?page=5">&gt;</a>
            </td>
        </tr>
    </tfoot>
    <tbody>
        <tr>
            <td>Lefevre</td><td>Hélène </td><td>65</td><td>Femme</td>
        </tr>
        <tr>
            <td>Pourcel</td><td>Marie-Roseline</td><td>34</td><td>Femme</td>
        </tr>
        <tr>
            <td>Allard</td><td>Remi </td><td>23</td><td>Homme</td>
        </tr>
        <tr>
            <td>Vanier</td><td>Gaëlle Sylvie</td><td>76</td><td>Femme</td>
        </tr>
        <tr>
            <td>Sardou</td><td>Tanguy</td><td>10</td><td>Homme</td>
        </tr>
    </tbody>
</table>
```

### Javascript

```js
<script type="text/javascript">
    $(function() {
        $("#usersGrid").asAjaxWebGrid({
            gridActionUrl: "@Url.Action("Index", "User")", 
            noResultContainerSelector: $("#noresults")
        });
    });
</script>
```

Voici un deuxième exemple d'utilisation avec une WebGrid MVC avec scénario un peu plus évolué (avec filtre) :

### Controller ASPNET

```csharp
public ActionResult Index(int? page, string sortFieldName, string sortDirection,
                                   int? sexFilter)
{
    int totalRowNumber;

    UserManager manager = new UserManager();
    IEnumerable<User> users = manager.GetUsers([...]);

    UsersListModel model = new UsersListModel
    {
        Page = page.HasValue ? page.Value : 0,
        RowPerPage = RowPerPage,
        SortDirection = sortDirection,
        SortFieldName = sortFieldName,
        TotalRowNumber = totalRowNumber,
        Users = users
    };

    if (Request.IsAjaxRequest())
    {
        return PartialView(model);
    }

    return View(model);
}
```

### Vue Razor

```csharp
<div>
    <input type="radio" name="sexFilter" value="0" id="menSexFilter" /> Homme
    <input type="radio" name="sexFilter" value="1" id="womenSexFilter" /> Femme
    <input type="button" id="filter" value="Filtrer" />
</div>

@{ 
    WebGrid grid = new WebGrid(sortFieldName: "sortFieldName",
        sortDirectionFieldName: "sortDirection",
        defaultSort: Model.SortFieldName,
        rowsPerPage: Model.RowPerPage);

    grid.Bind(Model.Users, autoSortAndPage: false, rowCount: Model.TotalRowNumber);

    if (Model.TotalRowNumber > 0)
    {
        grid.PageIndex = Model.Page > 0 ? Model.Page - 1 : 0;
    }

    grid.SortDirection = Model.SortDirection != "DESC" ? 
        SortDirection.Descending : 
        SortDirection.Ascending;
}

@{
    List<WebGridColumn> gridColumns = new List<WebGridColumn>
    {
        new WebGridColumn
        {
            ColumnName = "Name",
            Header = "Nom",
            CanSort = true
        },
        new WebGridColumn
        {
            ColumnName = "FirstName",
            Header = "Prénom",
            CanSort = true
        },
        new WebGridColumn
        {
            ColumnName = "Age",
            Header = "Age",
            CanSort = true
        },
        new WebGridColumn
        {
            ColumnName = "Sex",
            Header = "Sexe",
            CanSort = true,
            Format = @<text>@if (item.Sex == Sex.M) 
                            { <text>Homme</text> } 
                            else { <text>Femme</text> }
                      </text>
        }
    };
}

@if (Model.TotalRowNumber > 0)
{
    @grid.GetHtml(htmlAttributes: new { id = "usersGrid" }, columns: gridColumns)
}
else
{
    <div id="noresults">
        <span>Aucun résultat.</span>
    </div>
}
```

### HTML généré

```html
<table id="usersGrid">
    <thead>
        <tr>
            <th scope="col">
                <a href="/?sortFieldName=Name&amp;sortDirection=ASC">Nom</a>
            </th>
            <th scope="col">
                <a href="/?sortFieldName=FirstName&amp;sortDirection=ASC">Prénom</a>
            </th>
            <th scope="col">
                <a href="/?sortFieldName=Age&amp;sortDirection=ASC">Age</a>
            </th>
            <th scope="col">
                <a href="/?sortFieldName=Sex&amp;sortDirection=ASC">Sexe</a>
            </th>
        </tr>
    </thead>
    <tfoot>
        <tr>
            <td colspan="4">
                <a href="/?page=3">&lt;</a> 
                <a href="/?page=1">1</a> 
                <a href="/?page=2">2</a> 
                <a href="/?page=3">3</a> 
                4 
                <a href="/?page=5">5</a> 
                <a href="/?page=5">&gt;</a>
            </td>
        </tr>
    </tfoot>
    <tbody>
        <tr>
            <td>Lefevre</td><td>Hélène </td><td>65</td><td>Femme</td>
        </tr>
        <tr>
            <td>Pourcel</td><td>Marie-Roseline</td><td>34</td><td>Femme</td>
        </tr>
        <tr>
            <td>Allard</td><td>Remi </td><td>23</td><td>Homme</td>
        </tr>
        <tr>
            <td>Vanier</td><td>Gaëlle Sylvie</td><td>76</td><td>Femme</td>
        </tr>
        <tr>
            <td>Sardou</td><td>Tanguy</td><td>10</td><td>Homme</td>
        </tr>
    </tbody>
</table>
```

### Javascript

```js
<script type="text/javascript">
    $(function() {
        $("#usersGrid").asAjaxWebGrid({
            gridActionUrl: "@Url.Action("Index", "JQueryPluginAjaxWebGrid")",
            noResultContainerSelector: $("#noresults"),
            addCustomParameters: function() {
                var additionalParameters = {};
                additionalParameters["sexFilter"] = $("[name=sexFilter]").val();
                return additionalParameters;
            }
        });

        $("#filter").click(function() {
            if ($("#usersGrid").length == 0) {
                $("#noresults").data("ajaxWebGrid").refreshGrid();
            } else {
                $("#usersGrid").data("ajaxWebGrid").refreshGrid();
            }
        });
    });
</script>
```

## Fichier et package Nuget

Vous trouverez le fichier js ici : <a href="https://skydrive.live.com/redir?resid=2BAE7BB2DE0FBFC7!281" target="_blank">_sor.ajaxWebGrid.js_</a>

J'ai aussi créé un package nuget nommé AjaxWebGrid pour faciliter le déploiement dans vos projets VS : <a href="https://nuget.org/packages/AjaxWebGrid/" target="_blank">_AjaxWebGrid_</a>

N'hésitez pas à m'envoyer vos remarques.

**[Update : Une nouvelle version du plugin jQuery AjaxWebGrid est disponible. Pour plus d'informations : <a href="https://sebastienollivier.fr/blog/javascript/plugin-jquery-ajaxwebgrid-1-1-0/" target="_blank">https://sebastienollivier.fr/blog/javascript/plugin-jquery-ajaxwebgrid-1-1-0/</a>]**


