---
layout: post
title: 'Créer un WebJob en .NET Core 2 et le déployer via VSTS'
date: 2018-05-09
categories: azure
resume: 'Visual Studio ne propose pas encore de template pour créer des WebJob en .NET Core 2. Voyons comment faire et ensuite comment le déployer depuis VSTS.'
tags: webjob netcore2 vsts release azure
---
Microsoft ne propose pas à ce jour de template de projet permettant de créer des _WebJobs_ Azure en .NET Core. On va voir dans cet article comment transformer une application Console .NET Core en WebJob prêt à être déployé dans Azure.

# Création du WebJob

On va commencer par créer un projet _console_ via la commande suivante :

```js
dotnet new console
```

On va ensuite rajouter deux dépendances vers les packages nuget spécifiques aux WebJobs, qui nous permettront d'utiliser le SDK fournit par Microsoft :

```js
dotnet add package Microsoft.Azure.WebJobs
dotnet add package Microsoft.Azure.WebJobs.Extensions
dotnet add package Microsoft.Azure.WebJobs.ServiceBus
```

_Le package `Microsoft.Azure.WebJobs.ServiceBus` n'est nécessaire que si votre WebJob intéragit avec le ServiceBus._

Pour structurer notre WebJob, nous allons lui rajouter de l'injection de dépendance, un logger et nous allons utiliser le même mécanisme de configuration qu'ASPNET Core (via fichiers `appsettings.*.json`). Pour cela, il faut installer les packages nuget suivant :

```js
// Injection de dépendance
dotnet add package Microsoft.Extensions.DependencyInjection

// Log
dotnet add package Microsoft.Extensions.Logging
dotnet add package Microsoft.Extensions.Logging.Console

// Configuration
dotnet add package Microsoft.Extensions.Configuration
dotnet add package Microsoft.Extensions.Configuration.Json
dotnet add package Microsoft.Extensions.Options.ConfigurationExtensions
```

On peut maintenant créer une classe contenant le code de notre WebJob :

```csharp
public class Functions
{
    private readonly IProcessor processor;

    public Functions(IProcessor processor)
    {
        this.processor = processor;
    }

    public Task ProcessOnTimer([TimerTrigger("0 */5 * * * *", RunOnStartup = true)] TimerInfo timerInfo)
    {
        return this.processor.DoSomeProcessingStuff(0);
    }

    public Task ProcessQueueMessage([QueueTrigger("queueName")] int id)
    {
        return this.processor.DoSomeProcessingStuff(id);
    }
}
```

Cette classe déclare deux méthodes, `ProcessOnTimer` et `ProcessQueueMessage`, associées à des triggers. Notez la dépendance vers l'interface `IProcessor`, simplement pour illustrer le fonctionnement de l'injection de dépendances.

On va maintenant modifier la classe `Program` pour démarrer notre WebJob. La première étape consiste à charger la configuration :

```csharp
var configuration = new ConfigurationBuilder()
    .SetBasePath(Directory.GetCurrentDirectory())
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .Build();
```

On va ensuite créer le logger en utilisant la configuration précédente :

```csharp
var loggerFactory = new LoggerFactory()
    .AddConsole(configuration.GetSection("Logging:Console"));
```

Il faut maintenant initialiser notre injection de dépendances. On va donc instancier un `ServiceCollection` puis appeler une méthode `ConfigureServices` que nous implémenterons un peu plus tard.

```csharp
IServiceCollection services = new ServiceCollection();
ConfigureServices(configuration, services, loggerFactory);
```

Maintenant que la configuration est prête, il faut créer et démarrer le WebJob. Pour cela, on va s'appuyer sur la classe `JobHostConfiguration` :

```csharp
var hostConfiguration = new JobHostConfiguration();
hostConfiguration.LoggerFactory = loggerFactory;
hostConfiguration.Queues.MaxPollingInterval = TimeSpan.FromSeconds(10);
hostConfiguration.Queues.BatchSize = 1;
hostConfiguration.JobActivator = new InjectableJobActivator(services.BuildServiceProvider());
hostConfiguration.UseTimers();
hostConfiguration.UseServiceBus(new ServiceBusConfiguration
{
    ConnectionString = configuration["ServiceBus:ConnectionString"]
});

var host = new JobHost(hostConfiguration);
host.RunAndBlock();
```

A noter que l'on fournit une implémentation de `IJobActivator` utilisant notre container d'injection de dépendances (vous pouvez retrouver son implémentation directement sur le github du projet : <a href="https://github.com/sebastieno/webjobnetcore/blob/master/InjectableJobActivator.cs" target="_blank">`InjectableJobActivator.cs`</a>).

Revenons maintenant à la configuration de notre injection de dépendances. Il va falloir déclarer tous les éléments injectables, notamment la classe `Functions`, le logger et la configuration. Pour que notre exemple fonctionne, on va également déclarer une implémentation de `IProcessor`.

```csharp
private static void ConfigureServices(IConfiguration configuration, IServiceCollection services, ILoggerFactory loggerFactory)
{
    services.AddScoped<Functions>();
    services.AddScoped<IProcessor, NullProcessor>();
    services.AddSingleton(loggerFactory);
    services.AddSingleton(configuration);

    Environment.SetEnvironmentVariable("AzureWebJobsDashboard", configuration["AzureWebJobsDashboard"]);
    Environment.SetEnvironmentVariable("AzureWebJobsStorage", configuration["AzureWebJobsStorage"]);
}
```

A noter que l'on initialise deux variables d'environnement, `AzureWebJobsDashboard` et `AzureWebJobsStorage`, nécessaire pour le fonctionnement du WebJob.

Il reste maintenant à créer le fichier `appsettings.json` contenant la configuration de notre application :

```json
{
  "Logging": {
    "IncludeScopes": false,
    "Console": {
      "LogLevel": {
        "Default": "Warning"
      }
    }
  },
  "ServiceBus": {
    "ConnectionString": ""
  },
  "AzureWebJobsDashboard": "UseDevelopmentStorage=true",
  "AzureWebJobsStorage": "UseDevelopmentStorage=true"
}
```

Il ne faut pas oublier d'inclure ce fichier en tant que `Content`. Cela peut être fait en modifiant le fichier `csproj`en y rajoutant les informations suivants :

```xml
<ItemGroup>
  <None Remove="appsettings.json" />
</ItemGroup>

<ItemGroup>
  <Content Include="appsettings.json">
    <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
  </Content>
</ItemGroup>
```

Le WebJob est prêt. Nous allons voir comment le déployer dans Azure via VSTS.

_Vous pouvez retrouver les sources du WebJob sur <a href="https://github.com/sebastieno/webjobnetcore" target="_blank">GitHub</a>._

# Création de la build VSTS

La build Azure à créer est totalement standard :

![Build Azure](https://sebastienollivier.blob.core.windows.net/blog/creer-un-webjob-en-netcore2-et-le-deployer-via-vsts/build.png =219x224 "Build Azure")

On exécute dans l'ordre un `dotnet restore`, un `dotnet build` puis un `dotnet publish`. On publie ensuite les artifacts générés grâce à la tâche _Publish Artifact_.

# Création de la release VSTS

La release Azure est un peu plus spécifique. La raison vient dû fait que l'on va devoir déployer le WebJob dans un répertoire particulier d'un _App Service_.

Dans le cas des WebJob _triggered_, il va falloir déployer dans le répertoire `App_Data/jobs/triggered/<job_name>`, dans le cas des WebJob _continuous_, le répertoire sera `App_Data/jobs/continuous/<job_name>`.

![Release Azure](https://sebastienollivier.blob.core.windows.net/blog/creer-un-webjob-en-netcore2-et-le-deployer-via-vsts/release.png =269x112 "Release Azure")

La première tâche de la release va donc dézipper le contenu de l'artifact dans un répertoire portant l'arborescence cité précédemment. Dans le cas de notre WebJob, il s'agit d'un _continuous_ donc le _Destination folder_ sera : `$(System.DefaultWorkingDirectory)/mybuild/drop/webjobnetcore/App_Data/jobs/continuous/webjobnetcore`.

Ensuite, la tâche _Azure App Service Deploy_ va simplement déployer le répertoire dézippé dans l'_App Service_ souhaité. Dans notre cas, le répertoire à publier sera : `$(System.DefaultWorkingDirectory)/mybuild/drop/webjobnetcore`.

Et voilà, on a maintenant un WebJob qui tourne en net core 2 et déployé sur Azure.

Bon WebJobs !
