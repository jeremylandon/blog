---
title: "[Azure] Pattern Request/Response avec Azure Service Bus"
slug: "azure pattern request-response avec azure service bus"
date: 2020-03-22
description: ""
draft: false
tags:
  - dotnet
  - csharp
  - azure
  - pattern
series:
  - cloud
categories:
  - azure
image: images/logo/azure.png
---

Une des façons les plus simples et courantes d'utiliser les services de messagerie tels qu'Azure Service Bus est pour effectuer une communication unidirectionnel.
{{< img src="/images/content/200322-request-response-pattern-with-azure-service-bus/0.png" position="center" title="source" caption="https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview" alt="request" width="600px">}}

_Exemple: Bob en tant qu'envoyeur de message et Eve en tant que réceptionniste._
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;_1. Eve écoute la file d'attente Q1_
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;_2. Bob envoie un message sur Q1_
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;_3. Eve reçoit le message de Bob à partir Q1_

{{< boxmd >}}
**Mais comment Bob peut-il savoir que le traitement de Eve est terminé ?**
_Ou plus généralement comment ces 2 services peuvent-ils communiquer de manière bidirectionnelle?_
{{< /boxmd >}}

C'est là que le pattern **Request/Response (ou Request/Reply)** intervient. 

## Le pattern Request/Response

Ce pattern utilise un routage de message afin d'obtenir une communication bidirectionnel entre deux services.
Fonctionnellement c'est très simple, l'envoyeur envoie un message sur une file d'attente et attend une réponse sur une autre file d'attente.
{{< img src="/images/content/200322-request-response-pattern-with-azure-service-bus/1.png" position="center" title="source" caption="https://www.enterpriseintegrationpatterns.com/patterns/messaging/RequestReplyJmsExample.html" alt="request/response simple" width="400px">}}

Il existe 4 types de routage principaux qui sont très bien détaillés ici : [https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messages-payloads](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messages-payloads#message-routing-and-correlation)
Dans la version "la plus simple" (**Request/Response simple**), **il n'existe aucune garantie que Bob reçoit son message de réponse**. En effet si un nouvel utilisateur nommé Alice écoute la même file d'attente, elle pourrait intercepter le message destiné à Bob.
{{< img src="/images/content/200322-request-response-pattern-with-azure-service-bus/2.png" title="Concurrence entre Bob et Alice" position="center" alt="request/response simple" width="400px">}}
Naturellement une idée pourrait venir en tête assez rapidement : Bob créé une file d'attente à lui et attend la réponse dessus. Une fois la réponse obtenue il supprime la file d'attente.
C'est une bonne logique, **mais qui n'est pas à faire**, car:

- elle implique une gestion des services morts *(que faire si le processus crash sans avoir supprimé son service ? Doit-on supprimer le service avec les lettres mortes associées qui fournissent de précisieuses informations pour régler un potentiel bug ?...)*
- elle n'est évidemment pas performante *(il faudra à chaque fois créer et supprimer la queue, dans des fréquences parfois élevées)*
- elle introduit une pollution du Service Bus Namespace qui le rend difficile/impossible à analyser

Enfin c'est une solution personnalisée, et le standard est toujours à privilégier. N'en existe-t-il pas un ?
Justement si, grace au [protocole AMQP](https://docs.oasis-open.org/amqp/core/v1.0/os/amqp-core-messaging-v1.0-os.html) qui apporte une **notion de groupe**, retranscrite chez Microsoft sous le nom **session** qui résout très simplement et rapidement cette problématique.

## Les sessions (groupe)

Pour garantir que Bob soit le seul à récupérer le message de réponse qui lui est destiné, nous allons utiliser **les sessions** (ou plus précisement **les groupes du protocole AMQP**), qui pour faire simple apporte une multipléxage et ainsi plusieurs utilisateurs peuvent écouter la même file tout en ayant chacun leur propre groupe de message.
**Bob écoute et verrouille une session spécifique sur la file d'attente**, ainsi lui seul pourra écouter ses messages _(à l'image d'une sous file d'attente réservée à Bob)_.
Bob devra fournir des informations supplémentaires à son message afin que le réceptionneur (Eve) puisse envoyer correctement le message de réponse.
Techniquement cela se traduit par deux propriétés défini par le protocole AMQP 1.0 :

{{< notice warning >}}
le nom des champs du protocole AMQP ne sont pas forcement les mêmes que ceux de l'API _(ex: **group-id** dans le protocole = **SessionId** dans l'API)_
{{< /notice >}}

- **[ReplyTo](https://docs.microsoft.com/en-us/dotnet/api/microsoft.azure.servicebus.message.replyto?view=azure-dotnet)** (**reply-to**): chemin d'accès où Bob attend la réponse. Eve y enverra son message de réponse.
- **[ReplyToSessionId](https://docs.microsoft.com/en-us/dotnet/api/microsoft.azure.servicebus.message.replytosessionid?view=azure-dotnet)** (**reply-to-group-id**): l'id de la session écoutée par Bob. Eve assignera cette valeur à la propriété **SessionId** (**group-id**) de son message de réponse.

## Exemple

L'exemple sera réalisé avec [Microsoft.Azure.ServiceBus](https://www.nuget.org/packages/Microsoft.Azure.ServiceBus) et de deux file d'attente _(la logique reste la même avec les rubriques)_.
**Techniquement il est indispensable que la file d'attente de réponse accepte les sessions.**
Cela peut se fait via le portail Azure.
{{< img src="/images/content/200322-request-response-pattern-with-azure-service-bus/3.png" position="center" title="source" caption="https://docs.microsoft.com/en-us/azure/service-bus-messaging/message-sessions" alt="activate session" >}}

Ou via le code avec la propriété **[RequiresSession](https://docs.microsoft.com/en-us/dotnet/api/microsoft.servicebus.messaging.queuedescription.requiressession?view=azure-dotnet)**.

### Creation des files d'attente

Dans un premier temps nous allons créer pour la démonstration 2 files d'attente:

- **sample.request**: sans session, utilisé pour envoyer les messages de Bob à traiter par Eve.
- **sample.retry**: avec session, utilisé pour envoyer la réponse à Bob.

```c#
static async Task Main()
{
  var connectionString = "<your_connection_string>";
  var requestQueueName = "sample.request";
  var replyQueueName = "sample.reply";

  // create queues
  await Task.WhenAll(CreateQueueAsync(connectionString, requestQueueName, false), CreateQueueAsync(connectionString, replyQueueName, true));
}

// /!\ IT'S JUST FOR DEMO
static async Task CreateQueueAsync(string connectionString, string queueName, bool requiresSession)
{
  var managementClient = new ManagementClient(connectionString);
  if (await managementClient.QueueExistsAsync(queueName))
  {
      await managementClient.DeleteQueueAsync(queueName);
  }
  await managementClient.CreateQueueAsync(new QueueDescription(queueName)
  {
      RequiresSession = requiresSession
  });
}
```

Pour la démonstration chaque service sera simulé par un thread.

### Thread de l'envoyeur (Bob)

Bob envoie un message avec la propriété **ReplyTo** égal au chemin d'accès à la file d'attente de réponse et **ReplyToSessionId** égal à l'identifiant sa session.
Une fois le message envoyé Bob écoute sa session.

```c#

public static class SampleThreadFactory
{
...
  public static Thread CreateRequestor(string threadId, string connectionString, string requestQueueName, string replyQueueName)
  {
    return new Thread(async () =>
    {
      /*** send message ***/
      var messageSender = new MessageSender(connectionString, requestQueueName);
      var sessionId = "session-" + threadId;

      var message = new Message
      {
          MessageId = threadId,
          ReplyTo = new ServiceBusConnectionStringBuilder(connectionString) { EntityPath = replyQueueName }.ToString(),
          ReplyToSessionId = sessionId,
          TimeToLive = TimeSpan.FromMinutes(2)
      };

      Console.WriteLine($"{threadId} send a message to '{requestQueueName}' with replyToSessionId='{message.ReplyToSessionId}' and entityPath='{replyQueueName}'\n");
      await messageSender.SendAsync(message);
      await messageSender.CloseAsync();
      /*** send message ***/

      /*** wait response ***/
      SessionClient sessionClient = new SessionClient(connectionString, replyQueueName);
      var session = await sessionClient.AcceptMessageSessionAsync(sessionId);

      Console.WriteLine($"{threadId}'s waiting a reply message from '{replyQueueName}' with sessionId='{sessionId}'...\n");

      Message sessionMessage = await session.ReceiveAsync(TimeSpan.FromMinutes(2));

      Console.WriteLine($"{threadId} received a reply message from '{replyQueueName}' with sessionId='{sessionMessage.SessionId} (original: {sessionId})'\n");

      await session.CompleteAsync(sessionMessage.SystemProperties.LockToken);
      await session.CloseAsync();
      await sessionClient.CloseAsync();
      /*** wait response ***/
    });
  }
}
```

### Thread du répondeur (Eve)

Eve écoute la file d'attente sur laquelle Bob envoie ses messages.
Une fois qu'un message est intercepté, elle envoie un message de réponse sur le chemin d'accès défini par la propriété **ReplyTo**.
Le message envoyé contient la propriété **SessionId** définie à partir la propriété **ReplyToSessionId** du message intercepté _(et pour le suivi ainsi qu'une standardisation, il est fortement recommandé de définir la propriété **CorrelationId** avec l'id du message réceptionné)_.

```c#
public static class SampleThreadFactory
{
  public static Thread CreateReplier(string threadId, string connectionString, string requestQueueName)
  {
    return new Thread(() =>
    {
      var messageReceiver = new MessageReceiver(connectionString, requestQueueName);
      messageReceiver.RegisterMessageHandler(
        async (message, cancellationToken) =>
        {
          var connectionStringBuilder = new ServiceBusConnectionStringBuilder(message.ReplyTo);
          var replyToQueue = new MessageSender(connectionStringBuilder);
          var replyMessage = new Message(Encoding.UTF8.GetBytes($"processed by {threadId}"))
          {
              CorrelationId = message.MessageId,
              SessionId = message.ReplyToSessionId,
              TimeToLive = TimeSpan.FromMinutes(2)
          };

          /****  Simulate an action  *****/
          Console.WriteLine($"{threadId} works...\n");
          await Task.Delay(new Random().Next(1000, 2000), cancellationToken);
          /*******************************/

          Console.WriteLine($"{threadId} send a reply message to '{connectionStringBuilder.EntityPath}' with sessionId='{message.ReplyToSessionId}'\n");
          await replyToQueue.SendAsync(replyMessage);
        },        new MessageHandlerOptions(args => throw args.Exception)
        {
            MaxConcurrentCalls = 10
        });
    });
  }
```

### Initialisation du contexte de test

```c#
static async Task Main()
{
  var connectionString = "<your_connection_string>";
  var requestQueueName = "sample.request";
  var replyQueueName = "sample.reply";
  // create queues
  await Task.WhenAll(CreateQueueAsync(connectionString, requestQueueName, false), CreateQueueAsync(connectionString, replyQueueName, true));
  // start all
  Parallel.ForEach(new List<Thread>
  {
    SampleThreadFactory.CreateRequestor("REQUESTOR-BOB", connectionString, requestQueueName, replyQueueName),
    SampleThreadFactory.CreateReplier("REPLIER-EVE", connectionString, requestQueueName)
  }, (thread, state) => thread.Start());
  Console.Read();
}
```

## Résulat

![result-single](/images/content/200322-request-response-pattern-with-azure-service-bus/4.png)

**Et aucun problème avec plusieurs envoyeur, chacun reçoit le message de réponse qui lui est destiné**.

![result-multiple](/images/content/200322-request-response-pattern-with-azure-service-bus/5.png)

**Voilà!**

## Sources

### Repository

- [https://github.com/Golapadeog/sample-azure-service-bus-request-response](https://github.com/Golapadeog/sample-azure-service-bus-request-response)

### Documentation

- [https://www.enterpriseintegrationpatterns.com/patterns/messaging/RequestReplyJmsExample.html](https://www.enterpriseintegrationpatterns.com/patterns/messaging/RequestReplyJmsExample.html)
- [https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-amqp-request-response](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-amqp-request-response)
- [https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messages-payloads](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messages-payloads)
- [https://docs.microsoft.com/en-us/azure/service-bus-messaging/message-sessions](https://docs.microsoft.com/en-us/azure/service-bus-messaging/message-sessions)
- [https://en.wikipedia.org/wiki/Request%E2%80%93response](https://en.wikipedia.org/wiki/Request%E2%80%93response)
- [https://docs.oasis-open.org/amqp/core/v1.0/os/amqp-core-messaging-v1.0-os.html](https://docs.oasis-open.org/amqp/core/v1.0/os/amqp-core-messaging-v1.0-os.html)
