---
layout: post
title: 'Query Composer, gestion côté serveur'
date: 2014-12-17
categories: knockoutjs
resume: 'Après avoir créé le composant KnockoutJS, implémentant les Helpers ASPNET MVC permettant de faciliter et d''encapsuler son utilisation.'
tags: query-composer knockout-js
---
On a vu dans <a href="http://sebastienollivier.fr/blog/knockoutjs/query-composer">le post précédent</a> comment créer facilement un composant web permettant de composer des requêtes, en s'appuyant sur le Framework _KnockoutJS_ :

![Query Composer](https://sebastienollivier.blob.core.windows.net/blog/query-composer/querycomposer.png =914x202 "Query Composer")

Maintenant que le composant est disponible côté client, on va voir comment on peut développer des outils côté serveur permettant de faciliter son utilisation. 

## Helper Razor

Pour faciliter la déclaration du composant dans les vues Razor, nous allons créer un Helper MVC. La première étape va consister à créer un modèle .NET représentant le composant, de manière identique à ce qu'on avait fait pour _KnockoutJS_.

La classe `QueryComposer` représente donc ce composant. Il est identifié par un nom et contient une liste de champs.

```csharp
public class QueryComposer
{
    public string Name { get; internal set; }

    public List<FieldDefinition> Fields { get; internal set; }
}
```

La classe `FieldDefinition` représente un champ du composant. Un champ est caractérisé par un type, texte ou liste, par un libellé utilisé pour l'affichage ainsi que par un nom correspondant au nom de la propriété sur laquelle se fait le filtre. Si le champ est de type liste, il contiendra également une liste de valeurs.

```csharp
public class FieldDefinition
{
    public enum Types
    {
        Text,
        List
    }

    public string Text { get; set; }
    public string Name { get; set; }
    public Types Type { get; set; }
    public SelectList Values { get; set; }
}
```

Maintenant que le modèle est en place, on va créer une classe statique contenant une méthode d'extension sur la classe `HtmlHelper`, chargée de créer une nouvelle instance de la classe `QueryComposer` :

```csharp
public static class QueryComposerMvcHelper
{
    public static QueryComposer QueryComposer(this HtmlHelper helper, string name)
    {
        return new QueryComposer { Name = name, Fields = new List<FieldDefinition>() };
    }
}
```

On va également ajouter une méthode d'extension sur la classe `QueryComposer`, chargée d'exposer un constructeur de champs :

```csharp
public static class QueryComposerMvcHelper
{
    public static QueryComposer Fields(this QueryComposer component, Action<FieldDefinitionBuilder> fieldsBuilder)
    {
        fieldsBuilder(new FieldDefinitionBuilder(component));

        return component;
    }
}
```

La classe `FieldDefinitionBuilder` permet simplement de définir les champs du composant, en facilitant la syntaxe de déclaration. Son implémentation est la suivante :

```csharp
public class FieldDefinitionBuilder
{
    private readonly QueryComposer query;

    public FieldDefinitionBuilder(QueryComposer query)
    {
        this.query = query;
    }

    public void AddTextField(string name)
    {
        this.AddTextField(name, name);
    }

    public void AddTextField(string name, string text)
    {
        this.query.Fields.Add(new FieldDefinition { Name = name, Text = text, Type = FieldDefinition.Types.Text });
    }

    public void AddListField(string name, SelectList values)
    {
        this.AddListField(name, name, values);
    }

    public void AddListField(string name, string text, SelectList values)
    {
        this.query.Fields.Add(new FieldDefinition { Name = name, Text = text, Type = FieldDefinition.Types.List, Values = values });
    }
}
```

Enfin, une dernière méthode `Render` sera chargée de générer le code JavaScript et HTML nécessaire à l'initialisation du composant :

```csharp
public static class QueryComposerMvcHelper
{
    public static MvcHtmlString Render(this QueryComposer component)
    {
        var container = new TagBuilder("div");
        container.AddCssClass("query-composer");
        container.Attributes.Add("id", component.Name);
        container.Attributes.Add("data-bind", "template : { name: \'queryComposerTemplate\' }");

        StringBuilder jsBuilder = new StringBuilder();
        jsBuilder.AppendLine("<script type='text/javascript'>");

        jsBuilder.AppendLine("var fieldsDefinition = [");
        foreach (var field in component.Fields)
        {
            if (field.Type == FieldDefinition.Types.Text)
            {
                jsBuilder.AppendLine("new QueryComposer.Model.TextFieldDefinition('" + field.Name + "', '" + field.Text + "'),");
            }
            else if (field.Type == FieldDefinition.Types.List)
            {
                var data = string.Join(", ", field.Values.Select(v => "{ text: '" + v.Text + "', value: '" + v.Value + "'}"));

                jsBuilder.AppendLine("new QueryComposer.Model.ListFieldDefinition('" + field.Name + "', '" + field.Text + "', [" + data + "]),");
            }
        }

        jsBuilder.AppendLine("];");

        jsBuilder.Append("var vm = new QueryComposer.QueriesViewModel(fieldsDefinition);");
        jsBuilder.AppendLine("ko.applyBindings(vm, document.getElementById('" +        component.Name + "')[0]);");
        jsBuilder.Append("</script>");

        return MvcHtmlString.Create(container.ToString(TagRenderMode.Normal) + jsBuilder.ToString());
    }
}
```
Le Helper MVC est terminé. Voici un exemple de déclaration du composant :

```html
<form id="sampleForm>
    <h3>Critères de recherche :</h3>

    @(Html.QueryComposer("samplequery")
        .Fields(builder =>
        {
            builder.AddTextField("Title", "Titre");
            builder.AddListField("StatusId", "Statut", new SelectList(Model.Statuses, "Id", "Name"));
            builder.AddListField("IterationId", "Itération", new SelectList(Model.Iterations, "Id", "Name"));
            builder.AddListField("AreaId", "Zone", new SelectList(Model.Areas, "Id", "Name"));
        }).Render())

    <input type="submit" value="Rechercher">
</form>
```

Beaucoup plus simple à déclarer qu'en JavaScript :).

## Exécution des requêtes côté serveur

Une fois que l'utilisateur a saisi ses requêtes, il va falloir les exécuter. On va créer un Helper dont le rôle sera d'enrichir un `IQueryable` en fonction des requêtes saisies par l'utilisateur.

![Query Definition](https://sebastienollivier.blob.core.windows.net/blog/query-composer/query-definition.png =1098x188 "Query Definition")

L'idée de cet Helper, pour les requêtes saisies dans la capture précédente, est d'enrichir un `IQueryable` en ajoutant une clause `Where` ne récupérant que les éléments de l'_itération 2_ et en lien avec le _Backend_.

Pour récupérer les requêtes, saisies par l'utilisateur, côté serveur, il va falloir enrichir le modèle créé précédemment. Pour rappel, voici les `input` de type `hidden` générés par le composant :
```html
<input type="hidden" name="queries[0].type" value="0">
<input type="hidden" name="queries[0].field" value="Title">
<input type="hidden" name="queries[0].value">
<input type="hidden" name="queries[0].operator" value="&&">
```
Pour que le `ModelBinder` MVC récupère correctement les valeurs, on va créer un modèle reprenant la structure de ces `input`. La classe `QueryCompositionModel` représente la saisie de l'utilisateur. Elle contient une liste de requêtes.

```csharp
public class QueryCompositionModel
{
    public IEnumerable<Query> Queries { get; set; }
}
```

La classe `Query` représente une requête. Elle contient le type du champ associé, le nom de la propriété sur laquelle se fera le filtre, la valeur saisie par l'utilisateur et l'opérateur entre cette requête et la suivante.

```csharp
public class Query
{
    public FieldDefinition.Types Type { get; set; }

    public string Field { get; set; }

    public string Value { get; set; }

    public string Operator { get; set; }
}
```

La récupération des données saisies par l'utilisateur se fait maintenant via ce modèle :

```csharp
public async Task<ActionResult> Index(QueryCompositionModel model)
{
    […]
}
```

Le Helper va s'appuyer sur le modèle `QueryCompositionModel` pour enrichir l'`IQueryable`. Voici son code, tronqué pour plus de lisibilité (vous pouvez retrouver le code complet sur le GitHub : <a href="https://github.com/sebastieno/query-composer/blob/master/aspnetmvc.helpers/QueryableHelper.cs" target="_blank">QueryableHelper.cs</a>) :

```csharp
public static class QueryableHelper
{
    public static IQueryable<T> FilterByQueries<T>(this IQueryable<T> query, IEnumerable<Query> queries)
    {
        if(queries == null)
        {
            return query;
        }

        // Création d'un paramètre du type de l'entité liée aux requêtes
        var param = Expression.Parameter(typeof(T), "p");
        Expression body = null;

        // Supprimer les requêtes mal renseignées (pas de champ sélectionnée, pas de valeur renseignée)
        // Puis grouper par opérateur && et ||
        var groupedQueries = […];

        foreach (var group in groupedQueries)
        {
            Expression groupedBody = null;

            foreach (var queryModel in group)
            {
                MemberExpression property = null;

                // Récupération de la propriété de l'entité en fonction du champ sélectionné sur la requête
                var splittedFields = queryModel.Field.Split('.');
                foreach(var splittedField in splittedFields)
                {
                    if(property == null)
                    {
                        property = Expression.Property(param, splittedField);
                    }
                    else
                    {
                        property = Expression.Property(property, splittedField);
                    }
                }

                // Vérifier que le type du champ est un type simple
                […]
                
                // Génération d'une constante en fonction de la valeur renseignée sur la requête
                ConstantExpression value = null;
                if (property.Type == typeof(string))
                {
                    value = Expression.Constant(queryModel.Value);
                }
                else
                {
                    var convertedValue = Convert.ChangeType(queryModel.Value, property.Type);
                    value = Expression.Constant(convertedValue);
                }

               // Ajout de la condition à l'expression du groupe
                var subBody = Expression.Equal(property, value);
                if (groupedBody != null)
                {
                    groupedBody = Expression.AndAlso(groupedBody, subBody);
                }
                else
                {
                    groupedBody = subBody;
                }
            }

            // Ajout de l'expression du groupe à l'expression globale
            if (body != null)
            {
                body = Expression.OrElse(body, groupedBody);
            }
            else
            {
                body = groupedBody;
            }
        }

        // Ajout de l'expression à l'IQueryable de base
        if (body != null)
        {
            var subQuery = Expression.Lambda<Func<T, bool>>(body, param);

            query = query.Where(subQuery);
        }

        return query;
    }
}
```

Voici un exemple d'utilisation de cet Helper, dans une action POST d'un contrôleur :

```csharp
public async Task<ActionResult> Index(QueryCompositionModel model)
{
    try
    {
        using (var context = new Data.SampleDatabaseEntities())
        {
            var query = context.Tasks
                .Include(t => t.Area)
                .Include(t => t.Iteration)
                .Include(t => t.Status)
                .AsQueryable();
            
            query = query.FilterByQueries(model.Queries);

            var tasks = await query.ToListAsync();
            return PartialView("_GridResult", tasks);
        }
    }
    catch (Exception e)
    {
        throw new HttpException(400, e.Message);
    }
}

Et voilà, aussi simple que ça !

![Query execution](https://sebastienollivier.blob.core.windows.net/blog/query-composer/query-execution.png =1098x301 "Query execution")

Vous trouverez les sources du composant mises à jour sur le GitHub suivant : <a href="https://github.com/sebastieno/query-composer" target="_blank">https://github.com/sebastieno/query-composer</a>.

Il contient les scripts et le template du composant dans le répertoire knockoutjs, les Helpers MVC que l'on vient de créer dans le répertoire aspnetmvc.helpers, ainsi qu'une application ASP.NET MVC d'exemple, dans le dossier sample. Je vous invite à le tester et à le modifier selon vos besoins, voire même à contribuer si vous le souhaitez. Et évidemment, si vous avez des questions/remarques/optimisations, n'hésitez pas.

Bonnes exécutions de requêtes !

