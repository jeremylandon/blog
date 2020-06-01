---
title: "[Azure] Autoriser un agent de build hébergé à déployer sur Azure"
slug: "azure autoriser un agent de build heberge a deployer sur azure"
date: 2020-05-31
draft: false
tags:
  - powershell
  - azure devops
  - build
  - azure
  - network security group
  - nsg
series:
  - cloud
categories:
  - azure
image: images/logo/azure.png
---

**Azure DevOps** _(Travis, Google Cloud build & co)_ proposent des agents de build en mode Saas _(hébergés par un tier)_.
Ils sont très pratique car il n'y a aucune infrastructure à gérer mais ils peuvent poser problème lorsque la build doit se faire sur un service protégé.
En effet les agents étant public ils peuvent ne pas avoir les autorisations pour déployer sur un service.

J'ai rencontré le problème avec le service **[Azure Service Fabric](https://azure.microsoft.com/en-us/services/service-fabric/)** protégé par un **Network Security Group**, lors de l'exécution de la tâche de déploiement cette dernière ne se fait pas car la sécurité refuse toutes les ip public.

{{< notice info >}}
Si vous utilisez [ExpressRoute](https://azure.microsoft.com/en-us/services/expressroute/) le problème ne se pose pas car vous avez une liaison privée avec Azure.
{{< /notice >}}

## Solutions

### Première solution (qui sera la meilleure dans le futur)

Première solution mais qui n'est pas encore faisable aurait été d'ajouter le **Service Tag** **Azure Devops** au **Network Security Group**, **[mais ce dernier n'existe pas encore](https://devblogs.microsoft.com/devops/azure-devops-roadmap-update-for-2020-q2/)**.
Néanmoins il est possible d'ajouter le tag **AzureCloud** en attendant, mais c'est un "légèrement" bourrin…

### Deuxième solution (bourrin)

Seconde solution est d'ajouter les plages d'ip liées à votre service de build.
L'ensemble des ip public Azure est présente ici : **[https://www.microsoft.com/en-us/download/details.aspx?id=56519](https://www.microsoft.com/en-us/download/details.aspx?id=56519)** et tout les services fournissent ces données : **[travis](https://docs.travis-ci.com/user/ip-addresses/)**, **[google cloud](https://cloud.google.com/compute/docs/faq#find_ip_range)**...

{{< notice warning >}}
Ca marche mais c'est pas top, en effet les ip public peuvent changer _(la liste est mise à jour par Microsoft toutes les semaines)_, donc il va falloir penser à les mettre à jour et cela implique d'ajouter des plages d'ip pour _"gérer tous les cas"_.
{{< /notice >}}

### Troisième solution

Troisième solution que je préfère est d'ajouter lors de la build l'ip de l'agent de build au **Network Security Group**.
Sur **Azure Devops** cela se fait via la tâche **[Azure Cli](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/azure-cli?view=azure-devops)** et il existe des équivalents ailleurs (Jenkins: **[azure-cli](https://plugins.jenkins.io/azure-cli/)**, Bamboo: **azure-cli-run**…).
Voici le script générique _(à adapter en fonction du service de build utilisé et des ports requis)_:

{{< gist Golapadeog b418c74123daa8b2973e0078ed7aaa08 "add-inbound.ps1" >}}

On oublie pas de supprimer la règle après la build

{{< gist Golapadeog b418c74123daa8b2973e0078ed7aaa08 "delete-inbound.ps1" >}}

**Et voilà!**

## Sources

- [https://gist.github.com/Golapadeog/b418c74123daa8b2973e0078ed7aaa08](https://gist.github.com/Golapadeog/b418c74123daa8b2973e0078ed7aaa08)

### Documentation

- [https://azure.microsoft.com/en-us/services/expressroute/](https://azure.microsoft.com/en-us/services/expressroute/)
- [https://devblogs.microsoft.com/devops/azure-devops-roadmap-update-for-2020-q2/](https://devblogs.microsoft.com/devops/azure-devops-roadmap-update-for-2020-q2/)
- [https://www.microsoft.com/en-us/download/details.aspx?id=56519](https://www.microsoft.com/en-us/download/details.aspx?id=56519)
- [https://docs.travis-ci.com/user/ip-addresses/](https://docs.travis-ci.com/user/ip-addresses/)
- [https://cloud.google.com/compute/docs/faq#find_ip_range](https://cloud.google.com/compute/docs/faq#find_ip_range)
- [https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/azure-cli?view=azure-devops](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/azure-cli?view=azure-devops)
- [https://plugins.jenkins.io/azure-cli/](https://plugins.jenkins.io/azure-cli/)
