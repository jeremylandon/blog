---
title: "[Flutter] Comment valider une souscription avec in_app_purchase ?"
date: 2020-04-27
draft: false
tags:
  - flutter
  - dart
  - ios
  - android
  - mobile
series:
  - flutter
categories:
  - flutter
image: images/logo/flutter.png
---

J'ai eu la mauvaise surprise le mois dernier de voir tous mes abonnements sur android se faire rembourser automatiquement au bout de 3 jours.

Cela est dû à [Google Play Billing Library v2](https://developer.android.com/google/play/billing/billing_library_releases_notes) qui oblige à valider chaque souscription (_à l'image d'IOS_) dans les 3 jours, sinon => remboursement de l'utilisateur :money_with_wings:.

Cela est prévu depuis bien longtemps et l'équipe résponsable de la bibliothèque a très bien communiqué à ce sujet.
**MAIS** l'équipe de Flutter est à la traîne et la documentation porte vraiment à confusion, en effet il est indiqué que la validation n'est requise **que** pour IOS _(sous entendant qu'elle est automatiquement faite sur android)_.

{{< notice error >}}
if (Platform.isIOS) {
// Mark that you've delivered the purchase. <b>Only the App Store requires</b>
// this final confirmation.
InAppPurchaseConnection.instance.completePurchase(purchase);
}
{{< /notice >}}

**Mais rien n'est fait et il faudra en effet valider comme sur IOS chacune des souscriptions _(la documentation est donc actuellement fause...)_.**

{{< notice warning >}}
A l'écriture de l'article _(plus d'un mois après la mise en application de la règle des 3 jours)_ il est encore indiqué que la validation n'est requise que pour IOS…
{{< /notice >}}

Pour régler le problème il faudra avant tout de chose **passer sur une version >= 0.3.0 de [in_app_purchase](https://pub.dev/packages/in_app_purchase)** et tout simplement appeler la méthode **[completePurchase](https://pub.dev/documentation/in_app_purchase/latest/in_app_purchase/InAppPurchaseConnection/completePurchase.html)**.

```dart
if (purchaseDetails.pendingCompletePurchase) {
  await InAppPurchaseConnection.instance.completePurchase(purchaseDetails);
}
```

{{< notice info >}}
Inutile de vérifier la propriété [isAcknowledged](https://pub.dev/documentation/in_app_purchase/latest/billing_client_wrappers/PurchaseWrapper/isAcknowledged.html) _(pour éviter les requêtes inutiles)_, [la méthode le fait d'elle même](https://github.com/flutter/plugins/blob/master/packages/in_app_purchase/lib/src/in_app_purchase/google_play_connection.dart).

{{< /notice >}}

**Voilà!**

## Sources

### Documentation

- [https://github.com/flutter/plugins/blob/master/packages/in_app_purchase/lib/src/in_app_purchase/google_play_connection.dart](https://github.com/flutter/plugins/blob/master/packages/in_app_purchase/lib/src/in_app_purchase/google_play_connection.dart)
- [https://developer.android.com/google/play/billing/billing_library_releases_notes](https://developer.android.com/google/play/billing/billing_library_releases_notes)
- [https://pub.dev/documentation/in_app_purchase](https://pub.dev/documentation/in_app_purchase)
- [https://pub.dev/packages/in_app_purchase](https://pub.dev/packages/in_app_purchase)
