---
layout: post
title: 'Sécuriser un service WCF en utilisant des tokens SWT'
date: 2012-06-25
categories: federation-didentite
resume: 'Voyons comment ajouter le support des jetons SWT afin de sécuriser un service WCF.'
tags: wcf swt security
---
WIF (_Windows Identity Foundation_) permet de sécuriser un service WCF via le mécanisme de fédération d'identité en utilisant le standard SAML pour échanger les informations entre le STS et le service (pour plus d'informations : <a href="https://sebastienollivier.fr/blog/federation-didentite/service-wcf-claims-aware/" target="_blank">https://sebastienollivier.fr/blog/federation-didentite/service-wcf-claims-aware/</a>).

Par contre, WIF ne permet pas (encore ?) de sécuriser un service WCF en utilisant le standard SWT. On va voir dans ce post comment remédier à ce manque.

## Token SWT

Le format SWT (Simple Web Token) est un standard _Open Web Foundation Agreement Version 0.9_. Un token SWT contient un ensemble de paires clef / valeur, le tout étant encodé au format HTML.

Ci-dessous un exemple de token :

<font color="#2eb846">emailaddress=sebastien.ollivier%40gmail.com&name=Sebastien+OLLIVIER&http%3a%2f%2fschemas.microsoft.com%2faccesscontrolservice%2f2010%2f07%2fclaims%2fidentityprovider=Google&Audience=http%3a%2f%2flocalhost%3a4450%2fService.svc%2f&ExpiresOn=1340566492&Issuer=https%3a%2f%2fconnected-lab.accesscontrol.windows.net%2f</font>&<font color="#c60909">HMACSHA256=Q6WN2QrP%2bEnXrxek3%2fl0VuWqBzYtnpvTElgIQgKAUI8%3d</font>

Le token est composé de deux parties. La première partie (en vert ci-dessus) correspond à l'ensemble des informations transportées par le token, c'est-à-dire les claims. La deuxième partie (en rouge ci-dessus) correspond à la première partie hachée via la fonction SHA 256 HMAC.

L'émetteur et le destinataire du token partagent une clef privée permettant à l'émetteur de signer le token et au destinataire de vérifier l'authenticité du token.

## Custom ServiceAuthorizationManager

Pour sécuriser le service web, on va utiliser une brique WCF permettant de modifier la gestion des autorisations en créant une classe surchargeant `ServiceAuthorizationMananager` (plus d'informations sur la MSDN : <a href="http://msdn.microsoft.com/fr-fr/library/system.servicemodel.serviceauthorizationmanager.aspx" target="_blank">http://msdn.microsoft.com/fr-fr/library/system.servicemodel.serviceauthorizationmanager.aspx</a>).

```csharp
public class SimpleWebTokenAuthorizationManager : ServiceAuthorizationManager
{
    [...]
}
```

## CheckAccessCore

La méthode `CheckAccessCore` permet de vérifier si l'appelant est autorisé à contacter le service. C'est cette méthode qui va nous permettre de vérifier si un token SWT a été fourni, et si ce token est valide.

```csharp
protected override bool CheckAccessCore(OperationContext operationContext)
{
    [...]
}
```

La toute première étape va consister à vérifier si un token SWT est présent dans les headers de la requête WCF.

```csharp
protected virtual bool TryGetToken(out string token)
{
    token = null;

    WebHeaderCollection headers = 
        WebOperationContext.Current.IncomingRequest.Headers;
    string header = headers[HttpRequestHeader.Authorization]
        ?? headers["X-Authorization"];

    if (string.IsNullOrEmpty(header))
    {
        return false;
    }
    
    if (!header.StartsWith("WRAP access_token="))
    {
        return false;
    }

    string token = header.Substring("WRAP access_token=".Length);
    if (token[0] != '\"' || token[token.Length - 1] != '\"')
    {
        return false;
    }

    token = token.Substring(1);
    token = token.Substring(0, token.Length - 1);

    return true;
}
```

Il faut maintenant vérifier l'authenticité du token. L'idée est de hacher la première partie du token via la fonction SHA 256 HMAC en utilisant la clef privée et vérifier si on retrouve la même valeur que dans la deuxième partie du token.

```csharp
private bool IsHMACValid(string swt, byte[] sha256HMACKey)
{
    string[] swtWithSignature = swt.Split(new[] { "&HMACSHA256=" },
        StringSplitOptions.None);

    if ((swtWithSignature == null) || (swtWithSignature.Length != 2))
    {
        return false;
    }

    using (HMACSHA256 hmac = new HMACSHA256(sha256HMACKey))
    {
        byte[] signatureInBytes = 
            hmac.ComputeHash(Encoding.ASCII.GetBytes(swtWithSignature[0]));
        string signature = 
            HttpUtility.UrlEncode(Convert.ToBase64String(signatureInBytes));

        return signature == swtWithSignature[1];
    }
}
```

Il reste à lire le contenu du token pour vérifier que le token ne soit pas expiré, qu'il ait bien été émis pour notre service et par un STS de confiance.

```csharp
private bool IsAudienceTrusted(Dictionary<string, string> tokenValues)
{
    string audienceValue = tokenValues["Audience"];

    if (!string.IsNullOrEmpty(audienceValue))
    {
        return string.Equals(audienceValue, "http://localhost/MonService");
    }

    return false;
}

private bool IsIssuerTrusted(Dictionary<string, string> tokenValues)
{
    string issuerName = tokenValues["Issuer"];

    if (!string.IsNullOrEmpty(issuerName))
    {
        return string.Equals(issuerName, "https://dev.sts.fr/");
    }

    return false;
}

private bool IsExpired(Dictionary<string, string> tokenValues)
{
    string expiresOnValue = tokenValues["ExpiresOn"];
    ulong expiresOn = Convert.ToUInt64(expiresOnValue);
    
    TimeSpan currentEpochTime = 
        DateTime.UtcNow - new DateTime(1970, 1, 1, 0, 0, 0, 0);
    ulong currentTime = Convert.ToUInt64(currentEpochTime.TotalSeconds);

    if (currentTime > expiresOn)
    {
        return true;
    }

    return false;
}
```

Le manager `SimpleWebTokenAuthorizationManager` ressemble à :

```csharp
protected override bool CheckAccessCore(OperationContext operationContext)
{
    string token;
    if (TryGetToken(out token))
    {
        if (!IsHMACValid(token, Convert.FromBase64String("MA CLEF PRIVEE")))
        {
            return false;
        }

        Dictionary<string, string> values;
        try
        {
            // Récupération des paires sous forme d'un dictionnaire
            values = GetNameValues(token);
        }
        catch
        {
            return false;
        }

        if (IsExpired(values))
        {
            return false;
        }

        if (!IsIssuerTrusted(values))
        {
            return false;
        }

        if (!IsAudienceTrusted(values))
        {
            return false;
        }

        return true;
    }

    return false;
}

protected Dictionary<string, string> GetNameValues(string token)
{
    string[] splittedToken = token.Split('&');
    Dictionary<string, string> tokenValues = splittedToken.Aggregate(
        new Dictionary<string, string>(), (dico, values) =>
    {
        string[] splittedValues = values.Split('=');
        if (splittedValues.Length != 2)
        {
            throw new ArgumentException();
        }

        if (dico.ContainsKey(HttpUtility.UrlDecode(splittedValues[0])))
        {
            throw new ArgumentException();
        }

        dico.Add(HttpUtility.UrlDecode(splittedValues[0]), 
                 HttpUtility.UrlDecode(splittedValues[1]));

        return dico;
    });

    return tokenValues;
}
```

L'enregistrement se fait (via le fichier web.config) par la création d'un `service behavior`.
  
```xml
<serviceBehaviors>
    <behavior name="SwtBehavior">
        <serviceAuthorization serviceAuthorizationManagerType="Sor.SimpleWebTokenAuthorizationManager, Sor" principalPermissionMode="Custom" />
    </behavior>
</serviceBehaviors>
```

Le service est alors sécurisé via le mécanisme de fédération d'identité et attend un token au format SWT.

##Création de IClaimsIdentity

De la même manière qu'avec WIF, il faudrait que l'identité de l'appelant du service corresponde à celle transmise par le token.

Pour cela, il suffit de lire la première partie du token puis de créer un `IClaimsIdentity` en utilisant ses informations.

```csharp
private IClaimsIdentity CreateIdentity(Dictionary<string, string> values)
{
    ClaimsIdentity identity = new ClaimsIdentity("SWT");

    foreach (KeyValuePair<string, string> entry in values)
    {
        string claimType = entry.Key;

        if (claimType != "Issuer"
            && claimType != "ExpiresOn"
            && claimType != "Audience"
            && claimType != "HMACSHA256")
        {
            string[] parts = entry.Value.Split(',');

            foreach (string part in parts)
            {
                identity.Claims.Add(
                    new Claim(claimType, part,
                        ClaimValueTypes.String, "https://dev.sts.fr/")
                );
            }
        }
    }

    return identity;
}
```

On enregistre ensuite l'identité dans le contexte de l'opération dans la méthode `CheckAccessCore`.

```csharp
IClaimsIdentity identity = CreateIdentity(values);

IDictionary<string, object> properties = 
    operationContext.ServiceSecurityContext.AuthorizationContext.Properties;
IClaimsPrincipal principal = ClaimsPrincipal.CreateFromIdentity(identity);
properties["Principal"] = principal;
```

##Solution complète

Ci-joint vous trouverez un projet permettant de sécuriser un service WCF comme vu précédemment : 
<a href="https://skydrive.live.com/redir?resid=2BAE7BB2DE0FBFC7!285" target="_blank">_Sor.SwtSecurity.zip_</a>

Pour configurer le manager, il faut ajouter dans le fichier de configuration du service les appSettings suivant: 

* IssuerName : spécifie le nom du STS émettant les tokens SWT
* Audience : spécifie le nom de notre service
* SigningKey : clef privée utilisée pour hacher le token

Il suffit alors de renseigner le manager dans le fichier de configuration du service comme vu précédemment et le tour est joué.

