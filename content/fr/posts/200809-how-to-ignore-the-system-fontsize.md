---
title: "[Flutter] Ignorer la taille de la police du système"
slug: "flutter how to ignore the system fontsize"
date: 2020-08-09
draft: false
tags:
  - flutter
  - android
  - ios
  - mobile
series:
  - design
categories:
  - flutter
image: images/logo/flutter.png
---

En utilisant la librarie [flutter_html](https://pub.dev/packages/flutter_html) certains appareils avaient un problème dans la taille du texte, ce qui provoqué des overflow.
Le problème était en réalité présente partout mais cette librairie l'a mis en évidence.
Cela vient de la taille de la police du système. Certains appareils de grande taille (S20, Galaxy Note...) ont une taille de police par défaut "non standard" pour pallier à la taille de l'écran.

## Solution

La solution est assez simple, il faut pouvoir définir nous même le facteur d'agrandissent de la taille du texte.
Cela peut se faire avec la propriété **[textScaleFactor](https://api.flutter.dev/flutter/widgets/Text/textScaleFactor.html)** du composant **[Text](https://api.flutter.dev/flutter/widgets/Text-class.html)**.

```dart
@override
  Widget build(BuildContext context) {
    return MaterialApp(
      builder: (BuildContext context, Widget child) =>
        const Text('hello world!', textScaleFactor: 1.0),
    );
  }
```

Mais il est aussi possible d'appliquer une règle au niveau global en définissant cela sur le **[MediaQuery](https://api.flutter.dev/flutter/widgets/MediaQuery-class.html)**.

```dart
@override
Widget build(BuildContext context) {
  return MaterialApp(
    builder: (BuildContext context, Widget child) =>
      MediaQuery(
        data: MediaQuery.of(context).copyWith(textScaleFactor: 1.0),
        child: child,
      )
  );
}
```

{{< notice warning >}}
Attention à ne pas en abuser, en effet il faut prendre en compte que si l'utilisateur a choisi une taille de police c'est qu'il en a surement besoin pour son confort.
{{< /notice >}}

**Voilà !**

## Sources

### Documentation

- [https://api.flutter.dev/flutter/widgets/Text/textScaleFactor.html](https://api.flutter.dev/flutter/widgets/Text/textScaleFactor.html)
- [https://github.com/flutter/flutter/blob/master/packages/flutter/lib/src/widgets/text.dart](https://github.com/flutter/flutter/blob/master/packages/flutter/lib/src/widgets/text.dart)
