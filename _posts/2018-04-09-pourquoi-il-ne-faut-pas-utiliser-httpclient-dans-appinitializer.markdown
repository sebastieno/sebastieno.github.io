---
layout: post
title: 'Pourquoi il ne faut pas utiliser le HttpClient dans un APP_INITIALIZER ?'
date: 2018-04-09
categories: angular
resume: 'L''utilisation d''un intercepteur HTTP dans un APP_INITIALIZER peut provoquer des effets de bords. Comment peut''on s''en prémunir ?'
tags: angular httpclient appinitializer
---
`APP_INITIALIZER` est un token d'injection permettant de déclarer du code qui sera exécuté avant le boot de l'application Angular. 

Le scénario typique consiste à charger un fichier de configuration distant (pour ne pas avoir à l'inclure dans le bundle et pouvoir ainsi le modifier sans avoir à rebuilder / redéployer l'application). L'`APP_INITIALIZER` permet de différer le boot de l'application jusqu'au moment où la configuration est récupérée, permettant ainsi aux différents services d'y accéder en étant assuré que la configuration est prête.

Voici un exemple de code permettant de mettre en place ce mécanisme.

```typescript
export function configurationInit(config: ConfigurationService) {
	return () => config.init();
}

@NgModule({
 	providers: [{
		provide: APP_INITIALIZER,
		useFactory: configurationInit,
		deps: [ConfigurationService],
		multi: true
	};
]
```

_On crée une fonction que l'on associe au token `APP_INITIALIZER` permettant de déclencher le chargement de la configuration via le service `ConfigurationService`._

```typescript

@Injectable()
export class ConfigurationService {
	private settings: any;

	constructor(private httpClient: HttpClient) { }
	
	init(): Promise<any> {
		return this.httpClient.get('/config.json').toPromise().then(settings => {
			this.settings = settings;
			return settings;
		});
	}
	
	getConfiguration(key: string) {
		return this.settings[key];
	}
}
```

_Le service `ConfigurationService` expose une fonction `init` qui effectue un appel HTTP, via `HttpClient`, pour récupérer le fichier de configuration et initialiser une propriété privée._

## Alors, quel est le problème ?

Vous vous demandez sans doute quel est le problème puisque le code précédent fonctionne correctement. Voyons ça !

Dans la plupart des applications, vous aurez besoin d'un intercepteur HTTP, par exemple pour rajouter un header d'authentification. Voici un début d'implémentation pour illustrer :

```typescript
@Injectable()
export class AuthenticationHttpInterceptor implements HttpInterceptor {
	constructor(private injector: Injector) {
	}

	intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
		this.authService = this.injector.get(AuthenticationService);
		let token = this.authService.getAccessToken();
		
		[…]
	}
}
```

Et voici le service `AuthenticationService` permettant de récupérer le jeton :

```typescript

@Injectable()
export class AuthenticationService{
	constructor(private configurationService: ConfigurationService) { }
	
	[…]
}
```

Ce service est dépendant du `ConfigurationService`, notamment pour connaître l'URL de l'API à contacter.

Voici, dans l'ordre, ce qu'il va se passer :

1. Boot de l'application Angular
2. Exécution des `APP_INITIALIZER` et blocage du boot
3. Instanciation du `ConfigurationService` puis appel à la méthode `init`
4. Déclenchement de la requête HTTP via `HttpClient`
5. Exécution de l'intercepteur HTTP
6. Instanciation du service `AuthService` utilisé par l'intercepteur
7. Récupération du fichier de configuration et initialisation de la propriété `settings` du `ConfigurationService`
8. Fin du boot de l'application

Le problème vient de l'étape _6_. L'utilisation du service `AuthService` par l'intercepteur implique que ce service va être instancié lors de la phase de récupération de la configuration (dans l'`APP_INITIALIZER`) donc bien avant que l'application ne soit prête à booter. On peut se retrouver avec des effets de bord assez difficile à diagnostiquer, puisqu'il n'y a pas de lien directe entre l'`APP_INITIALIZER` et l'instanciation précoce du service.

## Comment je fais maintenant ?

La solution consiste à... ne pas utiliser le `HttpClient` en privilégiant le `XMLHttpRequest`:

```javascript
return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    xhr.open('GET', 'configurations/config.json');

    xhr.addEventListener('readystatechange', () => {
        if (xhr.readyState === XMLHttpRequest.DONE && xhr.status === 200) {
            this.settings = JSON.parse(xhr.responseText);
            resolve(this.settings);
        } else if (xhr.readyState === XMLHttpRequest.DONE) {
            reject();
        }
    });

    xhr.send(null);
});
```

On est assuré de ne plus utiliser l'intercepteur HTTP et donc d'éviter les problèmes évoqués précédemment.

Si on reprend les différentes étapes d'initialisation, cela donne maintenant :

1. Boot de l'application Angular
2. Exécution des `APP_INITIALIZER` et blocage du boot
3. Instanciation du `ConfigurationService` puis appel à la méthode `init`
4. Déclenchement de la requête HTTP via `XMLHttpRequest`
5. Récupération du fichier de configuration et initialisation de la propriété `settings` du `ConfigurationService`
6. Fin du boot de l'application

Bon boots d'applications.



