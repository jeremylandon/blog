---
title: "[Flutter] Exception en production avec flutter_local_notifications"
slug: "exception with flutter_local_notifications"
date: 2020-07-12
description: "J'ai eu la malchance de tomber sur cette exception qui ne se produit QUE sur les applications sur le store (internal test/beta/qa/prod... peu importe)."
draft: false
tags:
  - flutter
  - proguard
  - dart
  - exception
series:
  - exception
categories:
  - flutter
image: images/logo/flutter.png
---

{{< notice error >}}
**[ERROR:flutter/shell/platform/android/platform_view_android_jni.cc(39)] java.lang.AssertionError: AssertionError (GSON 2.8.5): java.lang.NoSuchFieldException: DrawableResource**
**06-13 23:55:46.181 30973 30973 E flutter : at d.d.d.q.a(:101)**
**06-13 23:55:46.181 30973 30973 E flutter : at d.d.d.q.a(:88)**
**06-13 23:55:46.181 30973 30973 E flutter : at d.d.d.q.a(:86)**
**06-13 23:55:46.181 30973 30973 E flutter : at com.dexterous.flutterlocalnotifications.d.e(:7)**
**06-13 23:55:46.181 30973 30973 E flutter : at com.dexterous.flutterlocalnotifications.d.a(:27)**
**06-13 23:55:46.181 30973 30973 E flutter : at com.dexterous.flutterlocalnotifications.d.a(:175)...**
{{< /notice >}}

J'ai eu la malchance de tomber sur cette exception qui ne se produit QUE sur les applications sur le store _(internal test/beta/qa/prod... peu importe)_.

La raison est simple, le package **[flutter_local_notifications](https://pub.dev/packages/flutter_local_notifications)** nécessite d'outrepasser les optimisations **[ProGuard](https://developer.android.com/studio/build/shrink-code)** sur son namespace.
La solution est donc d'indiquer cette contrainte à **ProGuard**.
Pour cela il suffit d'ajoutez dans le fichier **android/app/proguard-rules.pro** la ligne suivante :

```pro
-keep class com.dexterous.** { *; }
```

**voilà tout simplement!**

{{< notice info >}}
Faire attention au .gitignore: pour l'anecdote j'ai rencontré cette exception car le fichier **ProGuard n'était pas sous contrôle de source à cause de la configuration du .gitignore**, ce qui a empêché le serveur de build de réaliser des packages valides.
{{< /notice >}}

## Sources

### Documentation

- [https://developer.android.com/studio/build/shrink-code](https://developer.android.com/studio/build/shrink-code)
- [https://github.com/MaikuB/flutter_local_notifications/issues/452#issuecomment-577554022solution](https://github.com/MaikuB/flutter_local_notifications/issues/452#issuecomment-577554022solution)
