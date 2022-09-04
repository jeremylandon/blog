---
title: "[Postman] Réaliser un polling avec Postman"
slug: "postman realiser un polling avec postman"
date: 2020-06-15
draft: false
tags:
  - postman
  - javascript
  - test
  - quality
series:
  - test
categories:
  - postman
image: images/logo/postman.png
---

J'ai eu besoin à de nombreuses reprises de réaliser des polling dans mes **tests d'intégrations** avec [Postman](https://www.postman.com/). Je vous partage aujourd'hui les fonctions que j'utilise pour ça :beers:

{{< notice info >}}
**La logique sera stockée en variable globale afin de pouvoir être utilisée dans plusieurs requêtes.**
Cette technique permettant de partager des fonctions est détaillée sur le blog postman à l'adresse suivante : https://blog.postman.com/api-testing-tips-from-a-postman-professional/
{{< /notice >}}

Le code ci-dessous est à placer dans une requête "fake" _(ex: type Get sur **https://postman-echo.com/get**)_ dans la section **Pre-request Script**.
{{< gist jeremylandon 0494d9484c397beb39df3382cddb536d "postman-polling.js" >}}

La fonction de polling est utilisable dans la section **Test** de la requête sur lequel le polling sera réalisé :
{{< gist jeremylandon 0494d9484c397beb39df3382cddb536d "sample.js" >}}
Dans cet exemple la requête sera appelée tant que **le code HTTP de réponse ne sera pas 200**, avec **un délai de 1sec** entre chaque tentative et au **maximum 10 fois**.

{{< notice warning >}}
Pour que le polling fonctionne la requête devra être lancée via le **[Collection Runner](https://learning.postman.com/docs/postman/collection-runs/starting-a-collection-run/)**:
{{< /notice >}}

**Et voilà!**

## Sources

- [https://gist.github.com/jeremylandon/0494d9484c397beb39df3382cddb536d](https://gist.github.com/jeremylandon/0494d9484c397beb39df3382cddb536d)

### Documentation

- [https://blog.postman.com/api-testing-tips-from-a-postman-professional/](https://blog.postman.com/api-testing-tips-from-a-postman-professional/)
- [https://learning.postman.com/docs/postman/collection-runs/starting-a-collection-run/](https://learning.postman.com/docs/postman/collection-runs/starting-a-collection-run/)