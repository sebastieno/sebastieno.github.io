---
layout: post
title: 'Trucs et astuces, développement Windows Phone 8'
date: 2013-08-26
categories: wp8
resume: 'Voici une liste de trucs et astuces autour du développement Windows Phone 8 que j''ai construite lors du développement de l''application ''Remote Downloader''.'
tags: tips tricks wp8 windows-phone
---
Pour développer une application Windows Phone 8, Microsoft nous met à disposition un SDK (avec un émulateur) et nos outils de développement classiques (Visual Studio, Blend, TFS, etc.).

Bien que le SDK soit assez complet et bien fait, il y a quelques besoins simples qui ne sont pas résolvables de manière native. J'ai mis la liste de quelques besoins que l'on est amené à rencontrer régulièrement suivis à chaque fois de la solution qui me semblait être la meilleure (systématiquement trouvée sur des blogs ou des forums).

_Tous les points ci-dessous sont des besoins que j'ai rencontrés lors du développement de l'application <a href="http://sebastienollivier.fr/blog/wp8/remote-downloader" title="Remote Downloader" target="_blank">Remote Downloader</a>, cette liste n'est donc évidemment pas exhaustive._

Je vous laisse le soin de critiquer, compléter et/ou proposer de meilleures solutions.

## await sur un WebClient

Quand on veut faire une requête web en utilisant la méthode `DownloadStringAsync` de la classe `WebClient_` on ne peut pas faire de `await`, ce qui pourrait être extrêmement utile. Pour remédier à cela, il suffit d'encapsuler l'appel à cette méthode dans une <a href="http://msdn.microsoft.com/library/vstudio/dd449174.aspx" target="_blank" title="TaskCompletionSource">`TaskCompletionSource`</a> de la manière suivante :

```csharp
public static Task<string> DownloadStringAsync(this WebClient client, string url)
{
    var tcs = new TaskCompletionSource<string>();

    client.DownloadStringCompleted += (s, e) =>
    {
        if (e.Error == null)
        {
            tcs.SetResult(e.Result);
        }
        else
        {
            tcs.SetException(e.Error);
        }
    };

    client.DownloadStringAsync(new Uri(url));

    return tcs.Task;
}
```

L'appel se fait alors de la manière suivante :

```csharp
var client = new WebClient();
string result = await client.DownloadStringAsync("https://sebastienollivier.fr/blog");
```

On peut améliorer le comportement de la méthode en rajoutant un paramètre permettant de spécifier si l'on souhaite désactiver le cache (en rajoutant un paramètre dans la query string) :

```csharp
public static Task<string> DownloadStringAsync(this WebClient client, string url, bool disableCaching = false)
{
    var tcs = new TaskCompletionSource<string>();

    client.DownloadStringCompleted += (s, e) =>
    {
        if (e.Error == null)
        {
            tcs.SetResult(e.Result);
        }
        else
        {
            tcs.SetException(e.Error);
        }
    };

    if (disableCaching)
    {
        TimeSpan t = DateTime.UtcNow - new DateTime(1970, 1, 1);
        int secondsSinceEpoch = (int)t.TotalSeconds;

        if (url.Contains("?"))
        {
            url += "&nocache=" + secondsSinceEpoch;
        }
        else
        {
            url += "?nocache=" + secondsSinceEpoch;
        }
    }

    client.DownloadStringAsync(new Uri(url));

    return tcs.Task;
}
```

On pourrait imaginer une encapsulation encore plus générique qui permettrait de renvoyer un objet au lieu d'une chaîne de caractères :

```csharp
public static Task<T> DownloadAsync<T>(this WebClient client, string url)
{
    var tcs = new TaskCompletionSource<T>();

    client.DownloadStringCompleted += (s, e) =>
    {
        if (e.Error == null)
        {
            T deserializedResult = Helper.Deserialize<T>(e.Result);	
            tcs.SetResult(deserializedResult);
        }
        else
        {
            tcs.SetException(e.Error);
        }
    };

    client.DownloadStringAsync(new Uri(url));

    return tcs.Task;
}
```

La méthode `Helper.Deserialize` serait responsable de la déserialisation de la chaîne de caractères en `T`, en fonction du format de retour attendu (json, xml, etc.). L'appel se ferait alors de la façon suivante :


```csharp
var client = new WebClient();
Blog blog = await client.DownloadAsync<Blog>(url);
```

_Source : <a href="http://blog.galasoft.ch/archive/2013/01/27/using-asyncawait-with-webclient-in-windows-phone-8-or-taskcompletionsource.aspx" title="Source" target="_blank">http://blog.galasoft.ch/archive/2013/01/27/using-asyncawait-with-webclient-in-windows-phone-8-or-taskcompletionsource.aspx</a>_

_Note : vous pouvez aussi utiliser la classe `HttpClient`, qui répond (entre autres) au besoin évoqué, via le package Nuget suivant <a href="https://nuget.org/packages/Microsoft.Net.Http/2.1.10" target="_blank">https://nuget.org/packages/<wbr>Microsoft.Net.Http/2.1.10</a>_


## Transition entre deux pages et au changement d'orientation

Par défaut, quand on navigue entre deux pages WP8 ou quand on change l'orientation du device (de portrait à paysage et inversement), il n'y a pas de transition. Au moment où le changement a lieu, la page disparaît puis réapparaît dans sa nouvelle position. Ce comportement n'est pas forcément problématique, mais rajouter une petite animation donne une allure plus "pro", plus soignée à l'application.

Pour cela, on a à disposition le toolkit <a href="http://phone.codeplex.com/" target="_blank" title="Windows Phone Toolkit">The Windows Phone Toolkit</a> et la partie Navigation Transition_. Pour utiliser cette fonctionnalité, dans le fichier `App.xaml.cs`, il faut modifier le type de `RootFrame` :

```csharp
private void InitializePhoneApplication()
{
    if (phoneApplicationInitialized)
        return;

    // Create the frame but don't set it as RootVisual yet; this allows the splash
    // screen to remain active until the application is ready to render.
    RootFrame = new TransitionFrame();
    RootFrame.Navigated += CompleteInitializePhoneApplication;
```

Ensuite, pour chacune des pages concernées par les transitions, il faut ajouter le code XAML suivant :


```xml
<phone:PhoneApplicationPage
    xmlns:toolkit="clr-namespace:Microsoft.Phone.Controls;assembly=Microsoft.Phone.Controls.Toolkit">

[...]

<toolkit:TransitionService.NavigationInTransition>
    <toolkit:NavigationInTransition>
        <toolkit:NavigationInTransition.Backward>
            <toolkit:TurnstileTransition Mode="BackwardIn"/>
        </toolkit:NavigationInTransition.Backward>
        <toolkit:NavigationInTransition.Forward>
            <toolkit:TurnstileTransition Mode="ForwardIn"/>
        </toolkit:NavigationInTransition.Forward>
    </toolkit:NavigationInTransition>
</toolkit:TransitionService.NavigationInTransition>
<toolkit:TransitionService.NavigationOutTransition>
    <toolkit:NavigationOutTransition>
        <toolkit:NavigationOutTransition.Backward>
            <toolkit:TurnstileTransition Mode="BackwardOut"/>
        </toolkit:NavigationOutTransition.Backward>
        <toolkit:NavigationOutTransition.Forward>
            <toolkit:TurnstileTransition Mode="ForwardOut"/>
        </toolkit:NavigationOutTransition.Forward>
    </toolkit:NavigationOutTransition>
</toolkit:TransitionService.NavigationOutTransition>
```

On a maintenant de jolies transitions entre les pages.

_Source : <a href="http://phone.codeplex.com/" title="Source" target="_blank">http://phone.codeplex.com/</a>_

Le toolkit _The Windows Phone Toolkit_ ne permet pas d'animer les changements d'orientation. Pour cela, on peut utiliser la solution proposée ici : <a href="http://blogs.msdn.com/b/delay/archive/2010/07/13/spin-spin-sugar-updated-code-to-easily-animate-orientation-changes-for-any-windows-phone-application.aspx" target="_blank">http://blogs.msdn.com/b/delay/archive/2010/07/13/spin-spin-sugar-updated-code-to-easily-animate-orientation-changes-for-any-windows-phone-application.aspx</a>. Pour la mettre en place, il faut modifier, dans le fichier `App.xaml.cs`, le type de `RootFrame` : 


```csharp
private void InitializePhoneApplication()
{
    if (phoneApplicationInitialized)
        return;

    // Create the frame but don't set it as RootVisual yet; this allows the splash
    // screen to remain active until the application is ready to render.
    RootFrame = new HybridOrientationChangesFrame();
    RootFrame.Navigated += CompleteInitializePhoneApplication;
```

On a maintenant de jolies transitions aux changements d'orientation.

_Source : <a href="http://blogs.msdn.com/b/delay/archive/2010/07/13/spin-spin-sugar-updated-code-to-easily-animate-orientation-changes-for-any-windows-phone-application.aspx" title="Source" target="_blank">http://blogs.msdn.com/b/delay/archive/2010/07/13/spin-spin-sugar-updated-code-to-easily-animate-orientation-changes-for-any-windows-phone-application.aspx</a>_

Malheureusement, ces deux modifications sont incompatibles puisqu'elles nécessitent l'utilisation de deux types différents en tant que RootFrame. Pour combiner ces deux comportements, il faut modifier la classe `HybridOrientationChangesFrame` (du deuxième package) pour la faire hériter de `TransitionFrame` (du toolkit _The Windows Phone Toolkit_). De cette manière, on rajoute à la solution de transition aux changements d'orientation la possibilité d'effectuer des transitions aux changements de page.

```csharp
public class HybridOrientationChangesFrame : TransitionFrame
```

Maintenant, il faut utiliser `HybridOrientationChangesFrame` comme type de `RootFrame` et utiliser les noeuds `NavigationInTransition` et `NavigationOutTransition` dans la page pour définir les transitions de navigation.

Vous trouverez ci-joint la classe vue précédemment (il est nécessaire d'avoir installé le toolkit _The Windows Phone Toolkit_) : <a href="https://skydrive.live.com/redir?resid=2BAE7BB2DE0FBFC7!328" target="_blank">Sor.HybridOrientationChangesFrame</a>

## ApplicationBar & MVVM

Par défaut, il n'est pas possible de binder des commandes à l'`ApplicationBar`, cette dernière ne fonctionne qu'avec des évènements en code behind. Donc pas terrible en MVVM. Pour corriger cela, il existe un projet codeplex permettant de rajouter une couche de binding à l'`ApplicationBar` : <a href="http://bindableapplicationb.codeplex.com/" target="_blank" title="BindableApplicationBar">BindableApplicationBar</a>

Pour installer ce projet, vous pouvez utiliser le package nuget. Pour l'utiliser sur une page, il faut ajouter une référence vers `BindableApplicationBar` et utiliser la syntaxe suivante :

```xml
<phone:PhoneApplicationPage
    [...]
    xmlns:bar="clr-namespace:BindableApplicationBar;assembly=BindableApplicationBar">

[...]

<bar:Bindable.ApplicationBar>
    <bar:BindableApplicationBar IsVisible="True" IsMenuEnabled="True">
        <bar:BindableApplicationBar.MenuItems>
            <bar:BindableApplicationBarMenuItem Text="supprimer les tâches terminées" 
                Command="{Binding DeleteFinishedTasksCommand}" />
            <bar:BindableApplicationBarMenuItem Text="se déconnecter"
                Command="{Binding LogOffCommand}" />
            <bar:BindableApplicationBarMenuItem Text="a propos"
                Command="{Binding AboutCommand}" />
        </bar:BindableApplicationBar.MenuItems>
        <bar:BindableApplicationBarButton Text="rafraîchir" IconUri="/Images/appbar.refresh.png"
            Command="{Binding RefreshCommand}" />
        <bar:BindableApplicationBarButton Text="nouveau" IconUri="/Images/appbar.add.png"
            Command="{Binding NewDownloadTaskCommand}"/>
    </bar:BindableApplicationBar>
</bar:Bindable.ApplicationBar>
```

Au niveau du ViewModel, on retrouve des commandes classiques :

```csharp
public RelayCommand LogOffCommand { get; private set; }
public RelayCommand RefreshCommand { get; private set; }
public RelayCommand NewDownloadTaskViaBrowserCommand { get; private set; }
public RelayCommand NewDownloadTaskCommand { get; private set; }
public RelayCommand AboutCommand { get; private set; }
public RelayCommand<DownloadTaskModel> NavigateToTaskCommand { get; private set; }
public RelayCommand DeleteFinishedTasksCommand { get; private set; }
```

_Source : <a href="http://bindableapplicationb.codeplex.com/" title="Source" target="_blank">http://bindableapplicationb.codeplex.com/</a>_

## VisualState & MVVM

Rajouter quelques animations à l'application permet de la rendre plus sexy (afficher un message d'erreur en faisant glisser le message, etc.). Pour créer des animations, on doit passer par les <a href="http://msdn.microsoft.com/library/system.windows.visualstate.aspx" title="VisualState" target="_blank">`VisualState`</a>, en définissant l’apparence des contrôles quand ils sont dans un état spécifique.

Pour changer l'état d'un contrôle, il faut appeler la méthode `GoToState` de `VisualStateManager` en spécifiant le nom du contrôle ainsi que le nouvel état :

```csharp
VisualStateManager.GoToState("MyControlName", "LoadingState", true);
```

Quand on utilise le pattern MVVM, cette dépendance au nom du contrôle est gênante puisque le ViewModel n'est pas censé avoir connaissance de la vue. L'objectif serait ici de pouvoir binder l'état du contrôle à une propriété du ViewModel.

La solution consiste à créer un `DependencyObject` sur une propriété du ViewModel correspondant au `VisualState`. A chaque changement de valeur du `VisualState`, la nouvelle valeur sera appliquée au contrôle concerné en appelant le `VisualStateManager` : 

```csharp
public class StateManager : DependencyObject
{
    public static string GetVisualStateProperty(DependencyObject obj)
    {
        return (string)obj.GetValue(VisualStatePropertyProperty);
    }

    public static void SetVisualStateProperty(DependencyObject obj, string value)
    {
        obj.SetValue(VisualStatePropertyProperty, value);
    }

    public static readonly DependencyProperty VisualStatePropertyProperty =
        DependencyProperty.RegisterAttached("VisualStateProperty", typeof(string), typeof(StateManager),
        new PropertyMetadata((s, e) =>
        {
            var ctrl = s as Control;
            if (ctrl == null)
                throw new InvalidOperationException("This attached property only supports types derived from Control.");
            VisualStateManager.GoToState(ctrl, e.NewValue.ToString(), true);
        }));
}
```

Au niveau de la vue, le binding se fera comme ça (on ne peut binder que sur un élément de type `Control`, dans mon cas la page) :

```xml
<phone:PhoneApplicationPage
    [...]
    xmlns:helpers="clr-namespace:Synology.RemoteDownloadManager.WP8.Helpers"
    helpers:StateManager.VisualStateProperty="{Binding State}">
```

La propriété du ViewModel ressemble à :

```csharp
private DownloadTasksViewModelState state;
public DownloadTasksViewModelState State
{
    get { return state; }
    set
    {
        state = value;
        RaisePropertyChanged(() => this.State);
    }
}

public enum DownloadTasksViewModelState
{
    Normal,
    RefreshingTasks,
    AddingTask,
    Disconnecting,
    DeletingOldTasks,
    Error
}
```

_Source : <a href="http://tdanemar.wordpress.com/2009/11/15/using-the-visualstatemanager-with-the-model-view-viewmodel-pattern-in-wpf-or-silverlight/" title="Source" target="_blank">http://tdanemar.wordpress.com/2009/11/15/using-the-visualstatemanager-with-the-model-view-viewmodel-pattern-in-wpf-or-silverlight/</a>_

## Déterminer la couleur des icônes en fonction du thème du téléphone

Le possesseur d'un Windows Phone peut le personnaliser en choisissant notamment le thème d'arrière-plan (noir ou blanc). Si votre application utilise la couleur d'arrière-plan par défaut, une icône blanche ne sera pas visible si le thème est blanc et inversement pour le noir. Il est donc nécessaire de prévoir deux jeux d'icônes (icônes blanches et icônes noires) et switcher en fonction du thème. Cette solution n'est évidemment pas optimale.


Pour éviter ça, la solution consiste à appliquer un masque de la couleur inverse au thème d'arrière-plan sur une icône blanche. De cette manière, si l'arrière-plan est blanc, un masque noir sera appliqué et l’icône apparaîtra noire et inversement si l'arrière-plan est noir.


Pour chaque icône concernée par cette problématique, il faut appliquer la syntaxe XAML suivante :


```xml
<Rectangle Fill="{StaticResource PhoneForegroundBrush}" Width="39" Height="63" >
    <Rectangle.OpacityMask>
        <ImageBrush ImageSource="/Images/app.icon.white.png"/>
    </Rectangle.OpacityMask>
</Rectangle>
```

L'image est renseignée sous la forme d'un contrôle `ImageBrush` et un rectangle de la couleur `PhoneForegroundBrush` (inverse de la couleur de thème d'arrière-plan) permet d'appliquer le masque.

A partir du même code et de la même icône, on se retrouve avec les visuels suivants :

![Icone fond blanc](http://farm8.staticflickr.com/7376/9075635620_c7be40d2d0_o.png =319x85 "Icone fond blanc")

![Icone fond noir](http://farm3.staticflickr.com/2826/9075635624_e692b08b90_o.png =319x85 "Icone fond noir")

_Source : <a href="http://stackoverflow.com/questions/13972876/automatic-dark-light-icon-support-in-windows-phone-8" title="Source" target="_blank">http://stackoverflow.com/questions/13972876/automatic-dark-light-icon-support-in-windows-phone-8</a>_

## Justifier du texte

Pour justifier du texte, rien de plus simple. Il suffit de créer un `TextBlock` et de rajouter la propriété `TextAlignment="Justify"`. Sauf qu'aussi étonnant que cela puisse paraître, cela ne fonctionne pas. On aura une exception à l'exécution de l'application indiquant que la propriété `TextAlignment="Justify"` est incorrecte, même si Blend propose et  autorise cette valeur.

Pour contourner cela, on doit passer par un contrôle `RichTextBox` de la manière suivante :

```xml
<RichTextBox TextAlignment="Justify">
    <Paragraph>
        <Run Text="{Binding AboutPageResources.DataUsageContent, Mode=OneWay, Source={StaticResource LocalizedStrings}}"/>
    </Paragraph>
</RichTextBox>
```

_Source : <a href="http://www.rudyhuyn.com/blog/2011/11/08/comment-justifier-du-texte-sous-windows-phone/" title="Source" target="_blank">http://www.rudyhuyn.com/blog/2011/11/08/comment-justifier-du-texte-sous-windows-phone/</a>_

## Binder une enum à une liste

Quand on veut binder une enum à une liste en WPF, on peut passer par les _<a href="http://msdn.microsoft.com/library/system.windows.data.objectdataprovider.aspx" target="_blank">ObjectDataProvider</a>_ : <a href="http://msdn.microsoft.com/en-us/library/bb613576.aspx" target="_blank">http://msdn.microsoft.com/en-us/library/bb613576.aspx</a>. Malheureusement, Windows Phone ne nous donne pas accès aux ObjetDataProvider. Du coup, deux solutions sont possibles.

### Ajouter une propriété dans le ViewModel

La solution la plus simple consiste à rajouter une propriété au ViewModel correspondant à la liste des valeurs de l'enum :

```csharp
public OrderCriterion[] OrderCriteria
{
    get
    {
        return (OrderCriterion[])Enum.GetValues(typeof(OrderCriterion));
    }
}
```

Le binding se fait alors de manière standard.

### Passer par du code-behind

Si la solution précédente n'est pas acceptable, on va être obligé de passer par du code-behind. Voici un exemple de binding d'une enum sur un `ListPicker` :

```csharp
Array dateCriteria = Enum.GetValues(typeof(DateCriterion));
this.DateCriterionListPicker.ItemsSource = dateCriteria;
```

Si vous souhaitez en plus appliquer un binding, par exemple binder l'élément sélectionné à une propriété du ViewModel, il faudra utiliser la syntaxe suivante :

```csharp
this.DateCriterionListPicker.SetBinding(ListPicker.SelectedItemProperty, new Binding
{
    Mode = BindingMode.TwoWay,
    Path = new PropertyPath("Date")
});
```

Si vous souhaitez utiliser un converter sur les éléments bindés, on ne pourra pas passer par un binding (comme l'exemple précédent) parce que le converter serait appliqué à la liste entière, et non pas à chaque élément de la liste. La solution la plus simple serait d'appliquer le converter sur l'élément directement depuis un DataTemplate :

```xml
<toolkit:ListPicker x:Name="DateCriterionListPicker">
    <toolkit:ListPicker.ItemTemplate>
        <DataTemplate>
            <TextBlock Text="{Binding Converter={StaticResource DateCriterionToResourceConverter}}" />
        </DataTemplate>
    </toolkit:ListPicker.ItemTemplate>
</toolkit:ListPicker>
```

## Outils

En plus de ces points techniques, on peut avoir besoin d'outils pour réaliser ou compléter l'application. En dehors des outils classiques (Visual Studio, Blend, TFS), voici quelques outils qui pourraient vous être utiles lors de la création de l'application :

### Balsamiq

<a href="http://www.balsamiq.com/" target="_blank" title="Balsamiq">Balsamiq</a> est une application permettant de créer des mockups (prototypes d'interface utilisateur). L'avantage de cette application, au-delà du fait qu'il y a plein de composants graphiques et que son utilisation est très facile, est que les mockups ont des designs épurés (c'est à dire sans couleur, avec des contrôles simples, etc.) ce qui permet de se focaliser sur l'ergonomie et sur les interactions de l'application plutôt que sur la charte graphique.

Voici à quoi ressemble un exemple de mockups pour une application : 

![Balsamiq](https://farm4.staticflickr.com/3740/9152414582_fc4144587d_o.png =1017x710 "Balsamiq")

Cette application est payante, avec une version d'essai, mais il existe une démo online totalement fonctionnelle : <a href="http://builds.balsamiq.com/b/mockups-web-demo/" target="_blank" title="Balsamiq web demo">http://builds.balsamiq.com/b/mockups-web-demo/</a>. Avec cette version, vous ne pourrez pas profiter des features collaboratives fournies par balsamiq et vous aurez une popup qui s'affichera de temps en temps vous demandant si vous souhaitez passer sur une offre payante, mais vous pourrez créer, exporter et importer des mockups.

_Source : <a href="http://www.balsamiq.com/" title="Source" target="_blank">http://www.balsamiq.com/</a>_

### User Voice

Quand on crée une application, on souhaite souvent pouvoir recueillir les retours des utilisateurs, notamment les suggestions. Pour cela, il existe le site <a href="https://www.uservoice.com/" target="_blank" title="User Voice">User Voice</a> qui permet de créer un mini portail permettant aux utilisateurs de soumettre des idées. Un système de vote permet de visualiser les idées les plus plébiscitées, chaque utilisateur ayant 10 votes à distribuer entre les différentes idées, en pouvant en donner 3 maximum par idée.

![Portail User Voice](https://farm4.staticflickr.com/3774/9152396276_15345ca551_o.png =1140x495 "Portail User Voice")

Il existe plusieurs offres, dont une gratuite permettant d'avoir un accès aux fonctionnalités principales, ce qui est satisfaisant pour des petits projets.

Il n'existe pas encore d'intégration WP8 de User Voice, mais vous pouvez simplement ajouter un lien HTTP dans votre application dirigeant l'utilisateur vers votre User Voice.

_Source : <a href="https://www.uservoice.com/" title="Source" target="_blank">https://www.uservoice.com/</a>_

### BugSense</h3>

Quand on a une application déployée sur le store, comment avoir des logs quand l'application crash sur le téléphone des utilisateurs ? Soit on développe un mécanisme qui intercepte les `UnhandledException` et qui logge l'exception quelque part (WebService, Table Azure, etc.) soit on utilise un outil existant : <a href="https://www.bugsense.com/">Bug Sense</a>. Après s'être inscrit, avoir créé une application sur le portail de BugSense et avoir installé le SDK sur notre application (un package nuget et une ligne de code dans l'application), toute `UnhandledException` sera interceptée et les informations seront envoyées à BugSense. En retour, on recevra un mail contenant les informations loggées et on pourra retrouver ces informations sur le portail BugSense

Voici un exemple de mail d'erreur ainsi que le portail BugSense :

![Mail BugSense d'erreur](https://farm8.staticflickr.com/7348/9152396286_64a8a9468f_o.png =655x563 "Mail BugSense d'erreur")

![Portail BugSense](https://farm4.staticflickr.com/3708/9152396312_a13926496b_o.png =1181x585 "Portail BugSense")

Ici encore, il existe plusieurs offres dont une gratuite qui propose un accès aux fonctionnalités principales et un quota sur différentes fonctionnalités (rétention des erreurs limitées à 7 jours au lieu de 30 jours, etc.). Encore une fois, cette offre est assez complète pour être utilisée pour des petits projets.

Voilà pour ces petits trucs et astuces :), j'espère que ces informations pourront vous être utiles et n'hésitez pas à compléter / critiquer / proposer.


