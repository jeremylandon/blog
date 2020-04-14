---
title: "[Azure] Comment augmenter le nombre de conteneur sur l'émulateur Cosmos DB"
slug: "azure comment augmenter le nombre de conteneur sur lemulateur cosmos db"
date: 2020-04-12
description: "\"Sorry, we are currently experiencing high demand in this region, and cannot fulfill your request at this time. We work continuously to bring more and more capacity online, and encourage you to try again\""
draft: false
tags:
  - azure
  - Cosmos
  - exception
series:
  - exception
  - cloud
categories:
  - azure
image: images/logo/azure.png
---

Vous avez rencontré cette erreur en essayant de créer une nouveau conteneur sur l'émulateur local de Cosmos DB:
{{< notice error >}}
**"Sorry, we are currently experiencing high demand in this region, and cannot fulfill your request at this time. We work continuously to bring more and more capacity online, and encourage you to try again"**
{{< /notice >}}

## Augmenter le nombre de conteneur

Par défaut l'émulateur Cosmos DB ne gère que **25 conteneurs de taille fixe** ou **5 conteneurs illimités** _(ou un mixe des deux sachant qu'un conteneur de taille illimité équivaut à 5 conteneurs de taille fixe)_.
Il est possible d'augmenter cette limite jusqu'à **250 conteneurs de taille fixe** _(en acceptant de supprimer toutes ces collections)_, pour ce faire :

- **Quittez l'émulateur Cosmos**: cela prend plusieurs longues minutes => le plus rapide est d'arrêter les processus Cosmos DB comme un cochon :pig_nose: _(Azure Cosmos Master Service / Azure Cosmos Server Service / DocumentDB.GatewayService / Microsoft Azure Cosmos Emulator)_.
- **Supprimez les données de l'émulateur** en supprimant tous les fichiers de ce dossier: **%LOCALAPPDATA%\CosmosDBEmulator**.
- Lancer l'émulateur avec les **paramètre PartitionCount <= 250**.

```powershell
<# kill cosmos like a :pig_nose: #>
taskkill /im "CosmosDB.Emulator.exe" /f
taskkill /im "Microsoft.Azure.Cosmos.Server.exe" /f
taskkill /im "Microsoft.Azure.Cosmos.Master.exe" /f
taskkill /im "Microsoft.Azure.Cosmos.GatewayService.exe" /f

<# remove cosmos datas #>
Remove-Item "$env:LOCALAPPDATA\CosmosDBEmulator\*" -Recurse -Force

<# run cosmos with 250 partitions #>
& "C:\Program Files\Azure Cosmos DB Emulator\CosmosDB.Emulator.exe" /PartitionCount=250
```

**Voilà!** :ok_hand:

{{< notice info >}}
La limite du nombre de conteneur est mise en place pour limiter les ressources allouées à l'émulateur.
{{< /notice >}}

## Sources

### Documentation

- [https://stackoverflow.com/a/51617916/4181832](https://stackoverflow.com/a/51617916/4181832)
- [https://docs.microsoft.com/en-us/azure/Cosmos-db/local-emulator](https://docs.microsoft.com/en-us/azure/Cosmos-db/local-emulator)
