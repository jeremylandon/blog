---
title: "[Flutter] IOS에서 복사할 때 예외의 경우"
slug: "flutter throws an exception when trying to copy on ios"
date: 2019-10-21
description: 'Exception : The getter "pasteButtonLabel" was called on null'
draft: false
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

IOS에서 텍스트 범위를 복사/붙여넣기(길게 누름) 할 때, 아래의 메시지가 나타날 수 있습니다 :

{{< img src="/images/content/191021-flutter-throws-an-exception-when-trying-to-copy-on-ios/0.png" position="center" alt="The getter 'pasteButtonLabel' was called on null." >}}

{{< notice error >}}
**The getter 'pasteButtonLabel' was called on null.**
**Receiver: null**
**Tried calling: pasteButtonLabel**
{{< /notice >}}

이 경우는 IOS가 적합한 현지화 버튼을 찾지 못하는 것입니다.
이를 해결하기 위해선 개인의 IOS [LocalizationsDelegate](https://api.flutter.dev/flutter/widgets/LocalizationsDelegate-class.html) 만들면 됩니다.

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

그리고 MaterialApp에서 사용하십시오.

```dart {hl_lines=[6]}
MaterialApp(
     ...
      localizationsDelegates: const <LocalizationsDelegate<dynamic>>[
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
+       CupertinoLocalisationsDelegate(),
      ], ...
```

**Voilà!**
