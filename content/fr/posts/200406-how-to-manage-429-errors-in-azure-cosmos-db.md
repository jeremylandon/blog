---
title: "[Azure] Comment gérer les erreurs 429 sur Azure Cosmos DB"
slug: "azure comment gerer les erreurs 429 sur azure cosmos db"
date: 2020-04-06
description: ""
draft: false
tags:
  - azure
  - dotnet
  - csharp
  - performance
series:
  - cloud
categories:
  - azure
image: images/logo/azure.png
---

Les erreurs **HTTP 429** se produisent lorsque la consommation sur un conteneur est supérieure au débit provisionné.
{{< box >}}
Consommation en RU par seconde > RU provisionné par seconde = HTTP 429.
{{< /box >}}

**Avant tout de chose, est-ce grave ?**
Tout dépend de la situation, la finalité d'une erreur 429 est que la requête n'a pas été exécutée, cela peut être problématique dans des contextes où la cohérence des données est importante *(ex: ajout d'un profil utilisateur)*, mais moins si elle ne l'est pas *(ex: une vue sur une vidéo durant un pic de trafic de quelques secondes)*.

## Les réflexes à adopter

### Optimiser les requêtes

#### Optimiser les RU

Le **RU** *(unité de requête/request unit)* est l'unité de devise du débit. Elle est calculée via différents facteurs qui sont très bien détaillés sur [le site de Microsoft](https://docs.microsoft.com/en-us/azure/cosmos-db/request-units#request-unit-considerations).
{{< img src="/images/content/200406-how-to-manage-429-errors-in-azure-cosmos-db/0.png" position="center" title="source" caption="https://docs.microsoft.com/en-us/azure/cosmos-db/request-units" alt="request unit" >}}
Même en connaissant tout les facteurs il faut prendre en compte que la formule de calcul reste chez Microsoft, de ce fait il est très difficile de prédire à l'avance combien précisément une requête consomme en RU.
Néanmoins il est possible de connaître la consommation d'une requête après l'avoir exécutée et d'adapter en fonction en examinant l'en-tête **x-ms-request-charge** de la réponse.
Cette donnée est aussi disponible via le portail lors de l'exécution d'une requête:

![request charge](/images/content/200406-how-to-manage-429-errors-in-azure-cosmos-db/1.png)
Et directement via le SDK au travers de la propriété **[RequestCharge](https://docs.microsoft.com/en-us/dotnet/api/microsoft.azure.documents.client.resourceresponsebase.requestcharge?view=azure-dotnet)**.

```c#
var response = await conteneur.CreateItemAsync(foo, new PartitionKey(foo.Pk));
var ru = response.RequestCharge;
```

#### La fréquence

La fréquence de lancement des requêtes est primordiale et aussi la plus compliquée à déterminer *(nous n'allons ici pas parler des Bulk insert/update qui feront l'objet d'un autre article)*.

Supposons le cas où vous ayez 50 requêtes à 10 RU chacune, sur une provision à 400 RU/s le lancement de toutes les requêtes en parallèle provoquera des erreurs 429.
Deux solutions sont possibles : **augmenter temporairement le provisionnement** ou **lisser les requêtes dans le temps**.
Pour la deuxième solution cela demande de se poser la question suivante:
{{< alert info >}}
**Combien de RU puis-je consommer par seconde sans que cela ne provoque d'erreurs ailleur dans mon application ?**
{{< /alert >}}

Bien évidemment **il n'y a pas de réponse universelle à cette question**, tout dépend de votre contexte.
Mais une fois la réponse trouvée voici **[une méthode](https://gist.github.com/Golapadeog/7228a17b6287619f71ffd1ba60e4faa2)** qui permet de limiter le nombre de tâche en parallèle de manière temporisé.

{{< gist Golapadeog 7228a17b6287619f71ffd1ba60e4faa2 >}}

```csharp
var maxDegreeOfParallelism = 10;

await Enumerable.Range(0, 50).DelayParallelForEachAsync(async (index) =>
{
    var foo = Foo.Create(Guid.NewGuid().ToString());
    await conteneur.CreateItemAsync(foo, new PartitionKey(foo.Pk));
}, maxDegreeOfParallelism, 1000);
```

### Recommencer

**La solution la plus simple et courante est tout simplement de relancer la requête.**

En effet en partant du principe que le conteneur n'est pas sous-provisionné, des erreurs 429 peuvent arriver occasionnellement lors d'une activité à fort trafic.
Dans cette situation rien ne sert de sur-dimensionner le conteneur *(et ainsi payer plus)* pour éviter un problème qui intervient seulement quelques secondes dans la journée.

L'erreur HTTP 429 est accompagnée de l'en-tête **x-ms-retry-after-ms** qui indique dans combien de temps il sera possible de relancer la requête.
Heureusement cette logique de relancement est déjà gérée par le SDK via la propriété **[MaxRetryAttemptsOnRateLimitedRequests](https://docs.microsoft.com/en-us/dotnet/api/microsoft.azure.cosmos.cosmosclientoptions.maxretryattemptsonratelimitedrequests?view=azure-dotnet)**

{{< notice info >}}
Le nombre de nouvelles tentatives pour une requête est par **défaut de 9** sur le SDK.
{{< /notice >}}

```c#
CosmosClient cosmosClient = new CosmosClient(ConnectionString, new CosmosClientOptions()
{
    MaxRetryAttemptsOnRateLimitedRequests = 30
});
```

{{< alert warning >}}
**Super donc je n'ai qu'à mettre cette propriété à 1 000 000 et le problème est réglé !**
{{< /alert >}}

En théorie oui… **mais dans la quasi-totalité des cas c'est une très mauvaise idée**. En effet créer un très grand nombre de tentative :

- augmente le délai d'exécution des requêtes *(429 -> attendre 100ms -> 429 -> attendre 100ms -> […] -> OK)*
- peut provoquer des effets de bord très graves sur l'application *(lenteurs, threads bloqués, timeout…)*
- fait office de cache misère et ainsi masque le fait que votre conteneur est sous-provisionné

Concernant le nombre à mettre, il n'y a pas de “nombre magique” cela est à adapter en fonction du besoin *(Microsoft recommande néanmoins par exemple de passer cette valeur à 30 durant l'insertion d'un grand nombre d'entité)*.

### AutoPilot

En preview à l'écriture de cet article, Cosmos **[AutoPilot](https://docs.microsoft.com/en-us/azure/cosmos-db/autopilot-faq)** permet de mettre à l'échelle automatiquement le conteneur selon une plage de RU donnée.
![auto pilot](/images/content/200406-how-to-manage-429-errors-in-azure-cosmos-db/2.png)
Il faut néanmoins prendre en compte plusieurs aspects de **AutoPilot**:

- Il possède **sa propre tarification** *(plus élevée que le provisionné)*
- Il **n'empêche pas les 429** mais les limites fortement *(dans l'illustration si vous dépassé les 4000 RU/s les requêtes passeront en 429)*
- A l'écriture de l'article **il n'est pas encore possible de provisionner un conteneur en AutoPilot via le SDK** *([mais ceci est prévu](https://docs.microsoft.com/en-us/azure/cosmos-db/autopilot-faq#is-there-cli-or-sdk-support-to-create-containers-or-databases-with-autopilot-mode))*

### Augmenter le provisionnement

Si malgré toutes les actions précédentes des erreurs 429 apparaissent toujours et sont problématiques, **alors cela signifie simplement que votre conteneur est sous provisionnée par rapport à votre besoin**.
Pour aider à choisir le provisionnement adapté Microsoft met à disposition une calculatrice à capacité: **[https://cosmos.azure.com/capacitycalculator/](https://cosmos.azure.com/capacitycalculator/)**.

## Conclusion

Les erreurs HTTP 429 peuvent ne pas être problématiques en fonction de l'usage fait du conteneur. Elles doivent avant toute chose être un point d'alerte pour se poser des questions sur les performances du code/requêtes et le provisionnement des conteneurs.

## Sources

### Repository

- [https://gist.github.com/Golapadeog/7228a17b6287619f71ffd1ba60e4faa2](https://gist.github.com/Golapadeog/7228a17b6287619f71ffd1ba60e4faa2)

### Documentation

- [https://docs.microsoft.com/en-us/azure/cosmos-db/performance-tips](https://docs.microsoft.com/en-us/azure/cosmos-db/performance-tips)
- [https://docs.microsoft.com/en-us/azure/cosmos-db/provision-throughput-autopilot](https://docs.microsoft.com/en-us/azure/cosmos-db/provision-throughput-autopilot)
- [https://docs.microsoft.com/en-us/azure/cosmos-db/autopilot-faq](https://docs.microsoft.com/en-us/azure/cosmos-db/autopilot-faq)
- [https://docs.microsoft.com/en-us/azure/cosmos-db/optimize-cost-queries](https://docs.microsoft.com/en-us/azure/cosmos-db/optimize-cost-queries)
- [https://docs.microsoft.com/en-us/azure/cosmos-db/request-units](https://docs.microsoft.com/en-us/azure/cosmos-db/request-units)
- [https://cosmos.azure.com/capacitycalculator/](https://cosmos.azure.com/capacitycalculator/)
