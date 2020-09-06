---
title: "[NodeJs] Comment signer et vérifier un message avec Node.js"
slug: "nodejs how to sign and verify a message with node js"
date: 2020-09-06
draft: false
tags:
  - nodejs
  - javascript
  - security
series:
  - nodejs
categories:
  - tips
image: images/logo/nodejs.png
---

J'ai eu récement eu le besoin de pouvoir vérifier la signature retournée par une API le tout via Node.js.
Aujourd'hui je vous partage le code que j'utilise pour cela via un exemple et l'utilisation de la librarie **[node-rsa](https://github.com/rzcoder/node-rsa)**.

{{< gist Golapadeog c7601dec28a531ef0cefe71b200b0f1e "signandverify.js" >}}

Et vous pouvez testrer via un exemple **[Redpl.it](https://repl.it/@Golapadeog/sign-and-verify-signature-with-node-rsa)**

**Voilà !**

## Sources

### Documentation

- [https://github.com/rzcoder/node-rsa](https://github.com/rzcoder/node-rsa)
- [https://gist.github.com/Golapadeog/c7601dec28a531ef0cefe71b200b0f1e](https://gist.github.com/Golapadeog/c7601dec28a531ef0cefe71b200b0f1e)
- [https://repl.it/@Golapadeog/sign-and-verify-signature-with-node-rsa](https://repl.it/@Golapadeog/sign-and-verify-signature-with-node-rsa)
