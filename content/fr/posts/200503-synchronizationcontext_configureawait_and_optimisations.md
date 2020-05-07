---
title: "[.NET] SynchronizationContext, ConfigureAwait et optimisations"
slug: "dotnet synchronizationcontext configureawait et optimisations"
description: "Le SynchronizationContext permet d'applique une logique sur les opérations asynchrones et synchrones afin de s'adapter à un contexte voulu"
date: 2020-04-27
draft: false
tags:
  - dotnet
  - csharp
  - performance
series:
  - performance
categories:
  - dotnet
image: images/logo/dotnetcore.png
---

## Qu'est-ce qu'un SynchronizationContext ?

Le **[SynchronizationContext](https://docs.microsoft.com/en-us/dotnet/api/system.threading.synchronizationcontext)** permet d'applique une logique sur les opérations asynchrones et synchrones afin de s'adapter à un contexte voulu.

Par défaut la classe **SynchronizationContext** n'est qu'une base de travail, elle ne synchronise rien, elle expose entre autres une méthode virtuelle **[Post](https://docs.microsoft.com/en-us/dotnet/api/system.threading.synchronizationcontext.post)** qui a pour rôle de distribuer un message au contexte _(la "logique de synchronisation" se fera en grande partie ici)_.

Si on devait traduire ça en pseudo code cela donnerait quelque chose dans le style :

```csharp
async Task Foo()
{
  await Action1();
  Action2();
}
```

Equivaut à

```csharp
Task Foo()
{
  var task = Action1();
  var ctx = SynchronizationContext.Current;

  task.ContinueWith(task) => ctx.Post((o) => Action2(), null), TaskScheduler.Current);

  return task;
}
```

La méthode **Post** peut par exemple

- imbriquer l'action dans un lock pour rendre le tout thread-safe
- mettre en place une semaphore pour limiter le nombre de concurrence
- fournir des logs
- ...

bref c'est à adapter au besoin.

Il existe des implémentations spécifiques de **SynchronizationContext** utilisées nativement en **WPF**, **Winform** ou encore **ASP.NET** afin de s'adapter aux problématiques du framework cible.

Dans le cas d'**ASP.NET** l'implémentation est faite au travers de la classe interne **[AspNetSynchronizationContext](https://referencesource.microsoft.com/#system.web/AspNetSynchronizationContext.cs)**.

{{< notice warning >}}
On parle ici de **ASP.NET** et non de **ASP.NET Core** qui lui a **[abandonné l'utilisation du SynchronizationContext](https://devblogs.microsoft.com/dotnet/configureawait-faq/)**.
{{< /notice >}}

Cette implémentation est complexe mais on peut noter comme points notables qu'elle est utilisée lors de l'exécution du code d'une page, elle permet entre autres de capture le contexte http et de s'assurer que toutes les opérations asynchrones se terminant au même moment seront exécutées l'une après l'autre _(même si elles sont sur plusieurs threads différents)_.
Ce qui signifie que ceci :

```csharp
public async Task<ActionResult> Index()
{
  var myCollection = new List<int>(); // a non-thread-safe collection
  var tasks = new List<Task>();

  for (int i = 0; i < 100000; i++)
  {
    var myValue = i;
    tasks.Add(Task.Run(async () => {
      await Task.Delay(new Random().Next(5)); // simulate an action
      myCollection.Add(myValue); // add value to a non-thread-safe collection
    }));
  }

  await Task.WhenAll(tasks);

  return View();
}
```

Fonctionne parfaitement en ASP.NET même si la collection **[List](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1)** n'est pas thread-safe, le **[SynchronizationContext va effectuer un lock sur chacunes des opérations et ainsi il ne peut y avoir de concurrence sur l'ajout des éléments](https://referencesource.microsoft.com/#system.web/Util/SynchronizationHelper.cs,f0184c54fac66559)**.
En revanche en **ASP.NET Core**, sans ce **SynchronizationContext** il pourrait y avoir de la concurrence lors de l'ajout d'éléments dans la collection, et il sera necessaire de passer sur des collections thread-safe type **[ConcurrentBag](https://docs.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentbag-1)**.

## Que fait ConfigureAwait ?

**[ConfigureAwait](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.configureawait)** permet de spécifier si la suite du code doit s'exécuter dans le contexte d'origine ou non.

Par défaut cela est le cas et cela peut provoquer de gros problèmes.
Par exemple avec **WPF** où les actions seront réalisées en liaison avec le Thread UI, l'interface peut être bloquée/saccadée car les actions bloqueront de manière séquentiel le Thread UI pour capturer le contexte et continuer la suite du code.

Concrètement avec un **ConfigureAwait(false)** le **SynchronizationContext** n'est plus capturé.

```csharp
Debug.WriteLine(SynchronizationContext.Current != null); // true

await Task.Delay(10); // an action
Debug.WriteLine(SynchronizationContext.Current != null); // true

await Task.Delay(10).ConfigureAwait(false); // an action
Debug.WriteLine(SynchronizationContext.Current != null); // /!\ false
```

Cela implique qu'il n'est plus possible d'accéder aux données offertes par le contexte, par exemple en **ASP.NET** avec le **[HttpContext](https://docs.microsoft.com/en-us/dotnet/api/system.web.httpcontext.current)**

```csharp
Debug.WriteLine(System.Web.HttpContext.Current != null); // true

await Task.Delay(10); // an action
Debug.WriteLine(System.Web.HttpContext.Current != null); // true

await Task.Delay(10).ConfigureAwait(false); // an action
Debug.WriteLine(System.Web.HttpContext.Current != null); // /!\ false
```

Ce dernier n'est plus accessible comme tout le reste.

## Quand utiliser le ConfigureAwait(false) ?

Comme nous venons de le voir: **lorsque vous n'avez pas besoin du contexte**.
La majorité du temps le contexte est inutile et il convient de ne pas le récupérer. Il faudra donc penser à chaque action asynchrone appeler **ConfigureAwait(false)**.

{{< alert warning >}}
Il existe des façons bien moins lourdes pour gérer la non récupération du contexte que nous verrons dans un prochain post.
{{< /alert >}}

{{< boxmd >}}
**Je suis sous .NET Core, il n'y a pas de SynchronizationContext, je peux donc me passer de l'appel à ConfigureAwait**
{{< /boxmd >}}

En théorie oui, mais il arrive que non et ceux pour principalement 2 raisons:

- Si le code est prévu pour être utilisé aussi sur le **framework .NET** alors il convient de continuer à gérer les contextes
- Même si de base **.NET Core** n'implémente pas de **SynchronizationContext** rien n'empêche d'en implémenter un _(exemple Blazor en possède un)_.

La seule situation où cette "lourdeur" n'est pas à prendre en compte et sur vos propres applications où vous maitrisez l'existence/non existence du **SynchronizationContext**, et si la non gestion du contexte ne dégrade pas ou très peu les performances _(comme c'est le cas sur ASP.NET Core)_.

## Anecdote

Dans le cas où vous développez une librairie externe préparez vous à certains casses têtes "fonctionnels" :fire:.
J'ai voulu écrire ce post il y a plusieurs mois, lorsque j'ai vu ce code sur le net:

```csharp
public static async Task ForEachAsync<TEntity>(this IEnumerable<TEntity> entities, Func<TEntity, Task> action)
{
  foreach (var item in entities)
  {
    await action(item).ConfigureAwait(false);
  }
}
```

Comme son nom l'indique elle permet de simplement de faire un foreach async en inline.

**Quel est le problème ?**
Le problème est que cette méthode ne respecte pas selon moi un principe subjectif qui me tient à coeur qui est le **[POLS](https://en.wikipedia.org/wiki/Principle_of_least_astonishment)** _(principe de moindre surprise = une méthode fait ce qu'on pense qu'elle va faire)_.
Personnellement je m'attend à ce qu'elle fasse un simple foreach _(comme son nom et son existence même semble l'indiquer)_ mais elle traite aussi sur l'utilisation du contexte avec un comportement qui n'est pas celui par défaut.
Et voici ce qui peut arriver:

```csharp
await myHeaders.ForEachAsync(async s =>
{
  var userLang = System.Web.HttpContext.Current.Request.Headers[s]; // throw NullReferenceException on the second iteration
  await DoWorkAsync(userLang);
});
```

Comme le contexte n'est pas récupéré à la fin de la première itération, si l'action N°2 nécessite une donnée issue du contexte elle ne pourra pas la récupérer 🕵️‍♂️

Bon courage pour comprendre l'origine du problème si vous n'avez pas accès aux sources et sur des méthodes plus complexes... en bref il faut vraiment réflechir à si oui ou non notre méthode aurait besoin du contexte et si oui pouvoir donner la possibilité à l'utilisateur de l'utiliser ou non.

## Conclusion

Il y a encore énormément de choses à dire sur le **SynchronizationContext**, cela fera peut être l'objet d'autres posts, je vous laisse comme d'habitude quelques sources en fin de post pour aller un peu plus loin.
Avec la montée croissante de l'adoption d'**ASP.NET Core** les problématiques autour du **SynchronizationContext** sont de moins en moins primordiales.
Par exemple **[l'équipe en charge d'ASP.NET Core a fait le choix de ne plus le prendre en compte pour une bonne partie de leur code](https://devblogs.microsoft.com/dotnet/configureawait-faq/#comment-4247)** afin de gagner en lisibilité quitte à ce que cela induit une baisse de performance.
A noter que cette baisse de performance est extrêmement minime **[voir inexistante dû au simple fait qu'appeler ConfigureAwait(false) consomme des ressources](https://devblogs.microsoft.com/dotnet/configureawait-faq/#comment-4245)**.

## Sources

### Documentation

- [https://devblogs.microsoft.com/dotnet/configureawait-faq/](https://devblogs.microsoft.com/dotnet/configureawait-faq/)
- [https://blog.stephencleary.com/2017/03/aspnetcore-synchronization-context.html](https://blog.stephencleary.com/2017/03/aspnetcore-synchronization-context.html)
- [https://docs.microsoft.com/en-us/archive/msdn-magazine/2011/february/msdn-magazine-parallel-computing-it-s-all-about-the-synchronizationcontext](https://docs.microsoft.com/en-us/archive/msdn-magazine/2011/february/msdn-magazine-parallel-computing-it-s-all-about-the-synchronizationcontext)
- [https://referencesource.microsoft.com/#system.web/AspNetSynchronizationContext.cs](https://referencesource.microsoft.com/#system.web/AspNetSynchronizationContext.cs)
- [https://en.wikipedia.org/wiki/Principle_of_least_astonishment](https://en.wikipedia.org/wiki/Principle_of_least_astonishment)
