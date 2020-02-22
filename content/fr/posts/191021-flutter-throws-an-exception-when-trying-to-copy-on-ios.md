---
title: "[Flutter] Exception lors d'une copie sous IOS"
date: 2019-10-21
description: 'Exception : The getter "pasteButtonLabel" was called on null'
draft: false
hideToc: true
enableToc: false
enableTocContent: false
tocPosition: inner
tags:
  - flutter
  - dart
  - ios
  - mobile
  - exception
series:
  - exception
categories:
  - flutter
image: images/logo/flutter.png
---

Lors d'une tentative d'un copier/coller (pression longue) sur un champ Text sur IOS le message suivant peut apparaitre :

![The getter 'pasteButtonLabel' was called on null.](/images/content/191021-flutter-throws-an-exception-when-trying-to-copy-on-ios-1.png)

{{< notice error >}}
**The getter 'pasteButtonLabel' was called on null.**
**Receiver: null**
**Tried calling: pasteButtonLabel**
{{< /notice >}}

Si tel est le cas, c'est qu'IOS n'arrive tout simplement pas à trouver la localisation adaptée pour les boutons d'actions.
Pour corriger ce point il suffit de créer un [LocalizationsDelegate](https://api.flutter.dev/flutter/widgets/LocalizationsDelegate-class.html) personnalisé pour IOS :

```dart
class CupertinoLocalisationsDelegate extends LocalizationsDelegate<CupertinoLocalizations> {
  const CupertinoLocalisationsDelegate();

  @override
  bool isSupported(Locale locale) => true;

  @override
  Future<CupertinoLocalizations> load(Locale locale) => DefaultCupertinoLocalizations.load(locale);

  @override
  bool shouldReload(CupertinoLocalisationsDelegateold) => false;
}
```

Et de l'utiliser au niveau de la MaterialApp :

```dart {hl_lines=[6]}
MaterialApp(
     ...
      localizationsDelegates: const <LocalizationsDelegate<dynamic>>[
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
+       CupertinoLocalisationsDelegate(),
      ], ...
```

{{< notice info >}}
A noter qu'avec cette classe la localisation utilisée pour les actions sera celle par défaut du téléphone.
Pour corriger ce point il faut modifier la logique de la méthode **load(Local locale)** qui permet de définir la localisation.
{{< /notice >}}
