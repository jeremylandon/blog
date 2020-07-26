---
title: "[.NET] Comment vérifier les tokens Apple"
slug: "dotnet how to verify the apple tokens"
date: 2020-07-26
draft: false
tags:
  - dotnet
  - csharp
  - ios
  - apple
series:
  - apple
categories:
  - dotnet
image: images/logo/dotnetcore.png
---

## Validation

J'ai implémenté dans quelques applications **Flutter** le bouton **[Sign in with Apple](https://developer.apple.com/sign-in-with-apple/)**. Le processus d'authentication est classique et repose sur du **OAuth2** standard.
OAuth oblige Apple nous envoie des tokens JWT en guise de validation d'authentication des utilisateurs.
Une fois un token reçu **il est indispensable de le valider** pour éviter toute falsification.

Afin vérifier les tokens Apple met à disposition le endpoint **[https://appleid.apple.com/auth/keys](https://appleid.apple.com/auth/keys)** retournant un lot de clé public.
Pour trouver la bonne clé public il faut pour se référer au **kid** du token présent dans l'en-tête.

Pour faciliter les choses je vous propose une classe permettant de valider les tokens facilement.

## Solution/Code

```powershell
Install-Package System.IdentityModel.Tokens.Jwt
```

```csharp
var isValid = await new AppleTokenValidator().ValidateAsync(myToken);
```

{{< gist Golapadeog 130f1628e172cf090086da63d682f2b6 "AppleTokenValidator.cs" >}}

**voilà tout simplement!**

{{< notice info >}}
Je suis en train de réaliser une librarie pour faciliter l'utilisation de la fonctionnalité **Sign In With Apple** côté back (.NET). Cette fonctionnalité sera présente dans le package.
{{< /notice >}}

## Sources

- [https://gist.github.com/Golapadeog/130f1628e172cf090086da63d682f2b6](https://gist.github.com/Golapadeog/130f1628e172cf090086da63d682f2b6)

### Documentation

- [https://developer.apple.com/documentation/sign_in_with_apple/fetch_apple_s_public_key_for_verifying_token_signature](https://developer.apple.com/documentation/sign_in_with_apple/fetch_apple_s_public_key_for_verifying_token_signature)
- [https://oauth.net/articles/authentication/](https://oauth.net/articles/authentication/)
