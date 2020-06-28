---
title: "[Dev] Comment éviter les commits accidentels sur les fichiers de configuration"
slug: "dev how to prevent accidental commits on the config files"
date: 2020-06-28
draft: false
tags:
  - visual studio
  - dotnet
  - quality
  - msbuild
  - tips
series:
  - tools
categories:
  - tips
image: images/logo/visualstudio.png
---

Il n'est pas rare lors de développements à plusieurs que chacun ait besoin de sa propre configuration afin de pouvoir développer convenablement en local.
Le problème qui peut arriver et qu'un développeur commit par inadvertance un changement sur un fichier de configuration contenant ses propres données locals, cela pollue l'historique de fichier et pire, cela peut engendrer des problèmes chez les autres développeurs lors du prochain pull.

La technique que j'utilise qui a l'avantage de **fonctionner peu importe la technologie** est simplement d'utiliser un **template de configuration**.

Exemple dans le cas de Service Fabric, le fichier contenant l'ensemble des configurations du cluster peut être **Local.1Node.xml**.

La procédure consiste simplement à créer un fichier de référence **Local.1Node.template.xml** qui contiendra l'ensemble des paramètres par défaut et [d'ignorer sous git](https://git-scm.com/docs/gitignore) le fichier de configuration originel **Local.1Node.xml**.

{{< img src="/images/content/200628-how-to-prevent-accidental-commits-on-the-config-files/0.png" position="center" alt="config file template" >}}

A cela j'ajoute personnellement une tâche msbuild ayant pour objectif de copier le template si aucun fichier de configuration n'est présent avant la build *(plus de confort pour les prochains développeurs)*. Dans le **.csproj**, **.sfproj** ou autre il suffit d'ajouter cette tâche pre-build :

``` xml
<Project ...>
  ...
  <Target Name="BeforeBuild">
    <Copy SourceFiles="ApplicationParameters\Local.1Node.template.xml" DestinationFiles="ApplicationParameters\Local.1Node.xml" Condition="!Exists('ApplicationParameters\Local.1Node.xml')" />
  </Target>
  ...
</Project>
```

**voilà tout simplement!**
