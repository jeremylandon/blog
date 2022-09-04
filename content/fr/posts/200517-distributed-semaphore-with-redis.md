---
title: "[Redis] Implémentation d'une sémaphore distribuée en .NET"
slug: "redis implementation dune semaphore distribuee en dotnet"
description: "J'ai récemment eu besoin de limiter l'utilisation d'une portion de mon code entre différents services (à la manière d'une sémaphore)."
date: 2020-05-17
draft: false
tags:
  - dotnet
  - redis
  - csharp
  - performance
  - concurrence
series:
  - redis
categories:
  - dotnet
image: images/logo/redis.png
---

J'ai récemment eu besoin de limiter l'utilisation d'une portion de mon code entre différents services *(à la manière d'une sémaphore)*.
J'utilise **[RedLock.net](https://github.com/samcook/RedLock.net)** _(lock distribué)_ pour gérer la concurrence mais ce dernier ne fait qu'un verrou unitaire _(concurrence = 1)_ et donc ne répondait pas à mon besoin.

En parcourant un peu le site RedisLabs j'ai pu tomber sur **[cette algorithme](https://redislabs.com/ebook/part-2-core-concepts/chapter-6-application-components-in-redis/6-3-counting-semaphores/6-3-2-fair-semaphores/)** qui propose **une implémentation "juste"** _(premier arrivé premier servi)_ d'une semaphore distribuée.

N'existant pas d'implémentation en .NET j'ai du en développer une moi même.
Vous pouvez retrouver les sources ici: **[https://github.com/jeremylandon/DSemaphore.net](https://github.com/jeremylandon/DSemaphore.net)** *(le package nuget arrivera prochainement)*

## Utilisation

Dans un premier temps on créé une factory

```csharp
var connection = ConnectionMultiplexer.Connect("127.0.0.1:6379");
using (var semaphoreFactory = DSemaphoreFactory.Create(connection))
{
  // ...
}
```

Qui nous permet d'instancier nos sémaphores.
A la manière de la classe **[SemaphoreSlim](https://docs.microsoft.com/en-us/dotnet/api/system.threading.semaphoreslim)** on indique le maximum de concurrence que l'on souhaite

```csharp
int maxCount = 5;
await using (var semaphore = semaphoreFactory.CreateSemaphore("foo", maxCount))
{
    foreach (var entity in collection)
    {
       // ...
    }
}
```

Et il suffit de verrouiller avec la méthode **[WaitAsync](https://github.com/jeremylandon/DSemaphore.net/blob/master/src/DSemaphoreNet/IDSemaphore.cs)**.
{{< notice warning >}}
Veuillez à bien mettre un délai d'attente "cohérent". En effet si l'un de vos services s'arrête *(exemple: crash système)*, il ne pourra pas libérer explicitement son verrou et ce dernier ne sera considéré comme obsolète qu'à la fin du délai indiqué *(conséquence directe : il bloquera inutilement d'autres candidats)*.
{{< /notice >}}

```csharp
var timeout = TimeSpan.FromSeconds(3);
if (await semaphore.WaitAsync(timeout))
{
    // an action ...
}
```

**Et voilà c'est aussi simple que ça!** :ok_hand:

## Limites et recommandations

En implémentant l'algorithme proposée par RedisLabs j'ai pu y voir quelques limites _(qui sont propre au système distribué et difficilement corrigeables sans apporter de la lourdeur)_.

1. Comme énoncé précédemment si votre service s'arrête pour une raison X et donc n'a pas le temps de libérer son vérrouillage alors ce dernier ne sera considéré comme obsolète qu'à la fin du délai indiqué à l'appel de la méthode **[WaitAsync](https://github.com/jeremylandon/DSemaphore.net/blob/master/src/DSemaphoreNet/IDSemaphore.cs)**
2. La vérification fonctionne par polling, par défaut la fréquence est établie à 10ms, il est possible de la configurer à la création de la sémaphore. Mais cette fréquence provoque une incertitude sur la détection des verrous obsolètes. Il est recommandé d'utiliser des timeouts >= 1sec pour ne pas rencontrer de problème. *(pour faire simple la précision permettant de déterminer si une semaphore a expirée ou non est de +/- la fréquence de vérification)*

{{< boxmd >}}
**Cette solution remplace-t-elle une solution de Lock distribuée classique ?**
{{< /boxmd >}}

En théorie oui, car un lock peut être traité par une semaphore à 1.
En revanche je ne le recommande pas, même si cette implémentation est très performante, elle ne l'est pas autant que les solutions de lock classique qui ont bien moins de vérifications à faire.
**En bref c'est du code classique: semaphore pour limiter la concurrence et lock pour l'empêcher.**

{{< notice info >}}
A l'écriture de cet article, cette solution n'est qu'en béta, il reste notamment à séparer la logique entre le délai d'attente d'une acquisition et le TTL d'un verrou.
{{< /notice >}}

## Sources

- [https://github.com/jeremylandon/DSemaphore.net](https://github.com/jeremylandon/DSemaphore.net)

### Documentation

- [https://redislabs.com/ebook/part-2-core-concepts/chapter-6-application-components-in-redis/6-3-counting-semaphores/6-3-2-fair-semaphores/](https://redislabs.com/ebook/part-2-core-concepts/chapter-6-application-components-in-redis/6-3-counting-semaphores/6-3-2-fair-semaphores/)
- [https://docs.microsoft.com/en-us/dotnet/api/system.threading.semaphoreslim](https://docs.microsoft.com/en-us/dotnet/api/system.threading.semaphoreslim)
- [https://github.com/samcook/RedLock.net](https://github.com/samcook/RedLock.net)
