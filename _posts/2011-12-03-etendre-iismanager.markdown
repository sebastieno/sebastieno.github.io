---
layout: post
title: 'Etendre IIS Manager'
date: 2011-12-03
categories: iis
resume: 'Découvrons comment profiter des  mécanismes d''extensibilité de IIS pour créer un module IIS et y intégrer nos propres interfaces.'
tags: iis module
---
IIS 7 fournit un mécanisme d'extensibilité permettant aux développeurs d'intégrer leurs propres interfaces à IIS Manager. Ces interfaces sont appelés module IIS et sont exploités notamment par Web Platform Installer et par App Fabric.

![Web Platform installer](https://farm9.staticflickr.com/8066/8210263705_758e70da3a_o.png =294x127 "Web Platform installer")

<p style="text-align:center;"><sub>Icône du module Web Platform Installer</sub></p>

![IHM Web Platform installer](https://farm9.staticflickr.com/8204/8210263753_7015f0af84_o.png =413x194 "IHM Web Platform installer")

<p style="text-align:center;"><sub>IHM du module Web Platform Installer</sub></p>

![App Fabric](https://farm9.staticflickr.com/8206/8211352458_7ed33c2d7f_o.png =225x170 "App Fabric")

<p style="text-align:center;"><sub>Icônes du module AppFabric</sub></p>

![IHM App Fabric](https://farm9.staticflickr.com/8488/8210263675_6bd356b17f_o.png =423x257 "IHM App Fabric")

<p style="text-align:center;"><sub>IHM du module AppFabric</sub></p>

_Attention à ne pas confondre module IIS et module HTTP. Un Module IIS est une application qui va s'intégrer dans IIS Manager et donc étendre ses fonctionnalités. Un Module HTTP est une assembly qui est appelée à chaque requête effectuée sur une application web. Ces modules ont des rôles totalement différents._

Nous allons voir comment créer notre propre module IIS.

## Création du module IIS

Pour créer un module IIS, il est nécessaire de partir d'un projet de type _Class Library_. Rajoutons un `UserControl` de type `WinForm` qui fera office d'IHM du module.

La classe `ServerManager` de la dll `Microsoft.Web.Administration` (disponible dans le répertoire %WinDir%\System32\InetSrv) fournit un accès au système de configuration de IIS 7. Nous allons injecter une instance de `ServerManager` dans le constructeur dans le but d'accéder à ces informations et de pouvoir les modifier.

```csharp
public partial class ModuleUserControl : UserControl
{
    private ServerManager serverManager;
    public ModuleUserControl(ServerManager serverManager)
    {
        InitializeComponent();
        this.serverManager = serverManager;
    }
}
```

Nous allons ajouter deux propriétés au `UserControl` correspondantes au nom du site et au répertoire virtuel courant. Ces propriétés seront renseignées à chaque initialisation du module, c'est-à-dire à chaque fois que l'icône sera affichée dans IIS.

```csharp
private string siteName;
private string virtualPath;

public void Initialize(string siteName, string virtualPath)
{
    this.siteName = siteName;
    this.virtualPath = virtualPath;
}
```

De cette manière, on va pouvoir utiliser l'instance de `ServerManager` pour accéder à toutes les informations nécessaire, notamment concernant l'application courante. Le code suivant illuste l'accès à la liste des modules HTTP déclarés dans le fichier `applicationHost.config` pour l'application courante.

```csharp
Configuration applicationHostConfig = serverManager.GetApplicationHostConfiguration();
ConfigurationSection modulesSection = applicationHostConfig.GetSection
        ("system.webServer/modules", siteName + virtualPath);
```

Plus d'informations sur `ServerManager` ici :
<a href="http://msdn.microsoft.com/en-us/library/microsoft.web.administration.servermanager.aspx" target="_blank">http://msdn.microsoft.com/en-us/library/microsoft.web.administration.servermanager.aspx</a>

L'IHM du module maintenant prête, il va falloir créer le conteneur qui sera déployé dans IIS Manager. Ce conteneur est à deux niveaux. L'IHM sera contenu dans une page et la page sera contenue dans un module. C'est ce module qui sera alors déployé dans IIS.

Pour créer la page, il faut étendre la classe `ModulePage` de la dll `Microsoft.Web.Management` (disponible dans le répertoire %WinDir%\System32\InetSrv). Dans le constructeur de la page, il faut instancier le `UserControl` précédemment créé en lui donnant une instance de `ServerManager` et l'ajouter à la liste des contrôles de la page.

Il faut surcharger la méthode `OnActivated`. Cette méthode est déclenchée à chaque fois que le module est affiché dans IIS Manager et va nous permettre de renseigner les propriétés du `UserControl`.

```csharp
public class ExtensionModulePage : ModulePage
{
    private ModuleUserControl userControl;

    public ExtensionModulePage()
    {
        ServerManager serverManager = new ServerManager();
        userControl = new ModuleUserControl(serverManager);
        Controls.Add(userControl);
    }

    protected override void OnActivated(bool initialActivation)
    {
        base.OnActivated(initialActivation);
        if (initialActivation)
        {
            userControl.Initialize(Connection.ConfigurationPath.SiteName,
                  Connection.ConfigurationPath.ApplicationPath +
                  Connection.ConfigurationPath.FolderPath);
        }
    }
}
```

Pour créer le module, il faut étendre la classe `Module` (du namespace `Microsoft.Web.Management.Client` et non du namespace `System.Reflection`). La méthode `Initialize` va nous permettre d'enregistrer la page.

```csharp
public class CustomModule : Module
{
    protected override void Initialize(IServiceProvider serviceProvider,
              Microsoft.Web.Management.Server.ModuleInfo moduleInfo)
    {
        base.Initialize(serviceProvider, moduleInfo);
        IControlPanel controlPanel =
               (IControlPanel)GetService(typeof(IControlPanel));
        controlPanel.RegisterPage(new ModulePageInfo(this, typeof(CustomModulePage),
               "Mon module", "Mon module de test."));
    }
}
```

Le module est maintenant prêt. Il ne reste plus qu'à créer un provider en étendant `ModuleProvider`. L'objectif du provider est de fournir une définition du module à IIS.

```csharp
public class CustomModuleProvider : ModuleProvider
{
    public override ModuleDefinition GetModuleDefinition(IManagementContext context)
    {
        return new ModuleDefinition(Name,
                      typeof(CustomModule).AssemblyQualifiedName);
    }

    public override Type ServiceType
    {
        get { return null; }
    }

    public override bool SupportsScope(ManagementScope scope)
    {
        return true;
    }
}
```

## Déploiement du module IIS

La dll du projet doit être déployée dans le GAC. Il est donc nécessaire de signer le projet.

Une fois mise dans le GAC, il va falloir déclarer le module dans le fichier `administration.config` (disponible dans le répertoire %WinDir%\System32\inetsrv\config).

Ouvrez le fichier de configuration, localisez le nœud `moduleProviders` et rajoutez-y le module provider.

```xml
<add name="CustomIISModule" type="CustomIISModule.CustomModuleProvider,
     CustomIISModule, Version=1.0.0.0, Culture=neutral,
     PublicKeyToken=6d558c05d628f1ce" />
```

Pour déployer le module sur tous les sites IIS, rajoutez le dans le nœud `module` du nœud `location path="."`.

Pour le déployer sur un site spécifique, rajoutez le dans le nœud `module` du nœud `location path="<nom_site>"`.

```xml
<add name="CustomIISModule" /> 
```

Le module est alors déployé et apparait dans l'interface de IIS Manager.

![Icône du module](https://farm9.staticflickr.com/8344/8210263819_6741942257_o.png =686x630 "Icône du module")

![IHM du module](https://farm9.staticflickr.com/8350/8211352694_1b3b93a219_o.png =646x281 "IHM du module")

Si le module n'apparait pas dans IIS Manager, pensez à regarder les logs dans l'Event Viewer.

A bientôt.

