---
title: "[.NET] Comment réinitialiser une collection proprement: clear(), new() ou null ?"
slug: "dotnet comment reinitialiser une collection proprement clear new ou null"
date: 2020-04-12
description: "On m'a récemment posé la question sur quelle est la “meilleur” méthode pour supprimer les éléments d'une collection à taille variable."
draft: false
tags:
  - dotnet
  - csharp
  - performance
  - benchmark
series:
  - benchmark
categories:
  - dotnet
image: images/logo/dotnetcore.png
---

On m'a récemment posé la question sur "quelle est la "**meilleure**" méthode pour supprimer les éléments d'une collection à taille variable".

- myCollection.**Clear()**
- myCollection **= new List<string>()**
- myCollection **= null**

Selon moi comme toujours il n'y a pas de "**meilleure**" solution ou même de "**méthode magique**", <u>tout dépend de ce qu'on veut réellement faire et ce qu'on veut transmettre comme message aux prochains développeurs qui liront le code</u>.

{{< notice warning >}}
Nous ne traiterons pas le cas des collections **thread safe** de type ConcurrentBag & co. où la notion de concurrence rentre en jeu _(en excluant ce point la logique reste identique)_.
{{< /notice >}}

Avant toutes choses du point de vue de la performance et en prenant en critère la volonté de supprimer les éléments du tableau <u>sans aller dans la micro-optimisation</u>, les 3 méthodes peuvent être considérée comme équivalente = les éléments seront supprimés par le GC.

{{< boxmd >}}
**Alors peu importe la solution ça ne change rien ?**
{{< /boxmd >}}

**Non**, car même si la finalité sur les objets de la collection est la même, au globale le résultat est différent.

_Voici les classes utilisées dans les exemples, ces dernières ne sont utilisées que pour obtenir facilement un status visible de leur destruction._
{{< codes Foo TestList GetTestData >}}
{{< code >}}

```csharp
public class Foo
{
    private readonly int _id;

    public Foo(int id)
    {
        _id = id;
    }

    ~Foo()
    {
        Console.WriteLine($"Free Foo_{_id}");
    }
}
```

{{< /code >}}
{{< code >}}

```csharp
public class TestList<T> : List<T>
{
    ~TestList()
    {
        Console.WriteLine("Free TestList");
    }
}
```

{{< /code >}}

{{< code >}}

```csharp
private static TestList<Foo> GetTestData()
{
    var foos = new TestList<Foo>();
    for (var i = 0; i < 2; i++)
    {
        foos.Add(new Foo(i));
    }

    return foos;
}
```

{{< /code >}}
{{< /codes >}}

## null

**Assigner la collection à null revient à pousser un référence null et à l'assigner à notre variable.**

{{< alert info >}}
IL_00aa: **ldnull**
IL_00ab: **stloc.0**
{{< /alert >}}

```csharp
var foos = GetTestData();
foos = null;

GC.Collect();
GC.KeepAlive(foos);

// Free Foo_1
// Free Foo_0
// Free TestList
```

Cela signifie que notre collection initiale n'est plus rattachée aux GC Roots, elle sera donc considérée comme morte au prochain passage du GC.
Et comme cette dernière possède des objets rattachés qui n'ont pas d'autres références ils seront supprimés par la même occasion.
Donc cette méthode **supprime la collection et les objets rattachés**.

{{< notice warning >}}
Il est strictement inutile d'affecter un objet à null en fin de scope, en effet une fois le scope passé l'objet n'est de fait plus rattaché aux GC Roots _(s'il n'est pas référencé ailleurs)_.
{{< /notice >}}

```csharp
{
  var foos = GetTestData();
  foos = null;
}
```

Est identique à

```csharp
{
  var foos = GetTestData();
}
```

## new()

**Similaire à la méthode précédente à la différence que c'est une nouvelle instance de classe (=nouvelle référence) qui est poussée dans la pile d'évaluation et non null.**

{{< alert info >}}
IL_00aa: **newobj**
IL_00ab: **stloc.0**
{{< /alert >}}

```csharp
var foos = GetTestData();
foos = new TestList<Foo>();

GC.Collect();
GC.KeepAlive(foos);

// Free Foo_1
// Free Foo_0
// Free TestList
```

Comme avec **null**, la collection est supprimée dû au fait que sa référence n'est plus liée aux GC Roots et par effet de chaîne les objets rattachés aussi.

## Clear()

D'après la documentation cette méthode "**[supprime tous les éléments](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1.clear?view)**".
En vérité **[elle remplace les référence avec la collection](https://github.com/dotnet/runtime/blob/master/src/libraries/System.Collections/src/System/Collections/Generic/SortedList.cs), c'est le GC qui supprimera les objets de la mémoire.**
Le plus simple est de voir ce qui se passe en vrai.

```csharp
 var foos = GetTestData();
foos.Clear();

GC.Collect();
GC.KeepAlive(foos);

// Free Foo_1
// Free Foo_0
```

Contrairement aux méthodes précédentes ici on ne touche pas à la référence de la collection mais seulement aux références des objets de celle-ci.
Résultat **la collection étant toujours lié aux GC Roots, elle n'est pas libérée** mais comme les objets **Foo_1** et **Foo_2** ne sont plus liés à la collection ni rien d'autre **ils sont considérés comme mort par le GC** et ainsi se dernier les libère.
Aussi la collection reste la même de ce fait **ces propriétés acquises dont sa référence ou la [capacité](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1.capacity) reste inchangée**.

{{< notice info >}}
Pour rappel une **List** est simplement un tableau qui se redimensionne à la hausse automatiquement par réaffectation.
Par défaut la taille de ce tableau est de 4, si par l'accumulation de données le tableau est de taille 1000, il restera à 1000 après le passage de la méthode **Clear()**, cela évite de réallouer des espaces lors des 1000 prochains ajouts, mais cela implique que vous avez un tableau de taille 1000 en mémoire.
{{< /notice >}}

## Résumé

| méthode     | quand ?                                                                                                                                                                     |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **null**    | si vous souhaitez libérer la mémoire allouée par la collection et son contenu                                                                                               |
| **new()**   | si vous souhaitez libérer la mémoire allouée par la collection et son contenu tout en créant une nouvelle instance de la collection et implicitement une nouvelle capacité de base |
| **clear()** | si vous souhaitez libérer uniquement le contenu de la collection, en gardant les propriétés acquises de la collection                                                       |

## Conclusion

Il est important de bien utiliser chacunes des 3 méthodes avec pertinence <u>car elles aident à la compréhension du code</u>.
Micro-optimisation mis à part, **le code doit avant tout être compréhensible immédiatement**.

- si je vois **Clear()** je comprend de suite la volonté de supprimer le contenu
- si je vois **new List\<T\>(400)** je comprend directement la volonté de repartir sur une collection vierge de taille 400 car dans le contexte 400 est la bonne taille et que précédemment la collection était bien trop grosse
- si je vois **null** c'est que la collection ne sera plus utilisées plus tard dans le code et potentiellement que sa taille est problématique
- ...

Si on s'attarde sur la micro-optimisation _(on parle de seulement quelques Ticks...)_ un **new List\<T\>(x)** est plus performant qu'un **Clear()** car dans ce dernier un parcour du tableau est requis _(avec une compléxité O(n))_, <u>en revanche le message fourni aux prochains développeurs peut être ambiguë à la première lecture</u> _(en bref cela ne vaut pas le coup dans la quasi totalité des cas)_.

## Sources

### Documentation

- [https://en.wikipedia.org/wiki/Tracing_garbage_collection](https://en.wikipedia.org/wiki/Tracing_garbage_collection)
- [https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals)
- [https://github.com/dotnet/runtime/blob/master/src/libraries/System.Collections/src/System/Collections/Generic/SortedList.cs](https://github.com/dotnet/runtime/blob/master/src/libraries/System.Collections/src/System/Collections/Generic/SortedList.cs)
- [https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1.capacity](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1.capacity)
