---
title: "[Azure Devops] Générer les applications Flutter android"
slug: "azure devops generate android flutter apps"
date: 2020-08-23
draft: false
tags:
  - azure devops
  - flutter
  - android
  - mobile
series:
  - devops
categories:
  - azure devops
image: images/logo/azuredevops.png
---

Il est selon moi indispensable de passer la génération des applications au travers d'un système de build, pour le suivi, la praticité, la fiabilité et un gain de temps.
Aujourd'hui je vous partager le template de build que j'utilise pour générer les applications Flutter pour android.
Ce template utilise la **[tâche Azure Devops](https://github.com/aloisdeniel/vsts-flutter-tasks)** de **[aloisdeniel](https://github.com/aloisdeniel)**.

{{< gist Golapadeog 7b3f48f49bb636f9dd8edee3a553215f "generate-app-android.yml" >}}

**Voilà !**

## Sources

### Documentation

- [https://github.com/aloisdeniel/vsts-flutter-tasks](https://github.com/aloisdeniel/vsts-flutter-tasks)
- [https://gist.github.com/Golapadeog/7b3f48f49bb636f9dd8edee3a553215f](https://gist.github.com/Golapadeog/7b3f48f49bb636f9dd8edee3a553215f)
