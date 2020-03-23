---
title: "[Azure] Pattern Request/Response avec Azure Service Bus"
date: 2020-03-22
description: ""
draft: false
tags:
  - azure
  - pattern
series:
  - cloud
categories:
  - azure
image: images/logo/azure.png
---


Les services de messagerie comme Azure Service Bus sont la majorité du temps utilisés avec une communication unidirectionnel.
>*Exemple: Bob en tant qu'envoyeur de message et Eve en tant que réceptionniste.*
>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*1. Eve écoute la file d'attente Q1*
>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*2. Bob envoie un message sur Q1*
>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*3. Eve reçoit le message de Bob à partir Q1*

![request](/blog/images/content/200322-request-response-pattern-with-azure-service-bus/0.png)
source: [https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview)

{{< boxmd >}}
**Mais comment Bob peut-il savoir que le traitement de Eve est terminé ?**
{{< /boxmd >}}

C'est là que le pattern Request/Response (ou Request/Reply) intervient. Ce pattern utilise un routage de message afin d'obtenir une communication bidirectionnel entre deux services.

Fonctionnellement c'est très simple, l'envoyeur envois un message sur une file d'attente et attend une réponse sur une autre file d'attente.
![request/response simple](/blog/images/content/200322-request-response-pattern-with-azure-service-bus/1.png)
source: [https://www.enterpriseintegrationpatterns.com/patterns/messaging/RequestReplyJmsExample.html](https://www.enterpriseintegrationpatterns.com/patterns/messaging/RequestReplyJmsExample.html)

Il existe 4 types de routage principaux qui sont très bien détaillés ici : [https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messages-payloads](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messages-payloads#message-routing-and-correlation)
Dans la version la plus simple (Request/Response simple). Il n'existe aucune garantie que Bob reçoit son message de réponse. En effet si un nouvel utilisateur nommé Alice écoute la même file d'attente, elle pourrait intercepter le message destiné à Bob.
![request/response concurrence](/blog/images/content/200322-request-response-pattern-with-azure-service-bus/2.png)

## Request/Response multiplexé

Pour garantir que Bob soit le seul à récupérer le message de réponse qui lui est destiné, nous allons utiliser les sessions.
Bob écoute et verrouille une session spécifique sur la file d'attente, ainsi seul lui pourra écouter ces messages (à l'image d'une sous file d'attente réservée à Bob).
Technique Bob devra fournir des informations supplémentaires à son message afin que le réceptionneur puisse envoyer correctement le message de réponse.
Techniquement cela se traduit par deux propriétés :

- ReplyTo: chemin d'accès où Bob attends la réponse. Eve enverra son message de réponse sur ce chemin d'accès
- ReplyToSessionId: l'id de la session écoutée par Bob. Eve assignera cette valeur à la propriété SessionId de son message de réponse.

## Exemple

L'exemple sera réalisé à partir du framework Microsoft.Azure.ServiceBus ([https://www.nuget.org/packages/Microsoft.Azure.ServiceBus](https://www.nuget.org/packages/Microsoft.Azure.ServiceBus)) et de deux file d'attente (la logique reste la même avec les rubriques).
Techniquement il est indispensable que la file d'attente de réponse accepte les sessions.
Cela peut se fait via le portail Azure
![activate session](/blog/images/content/200322-request-response-pattern-with-azure-service-bus/3.png)
source: [https://docs.microsoft.com/en-us/azure/service-bus-messaging/message-sessions](https://docs.microsoft.com/en-us/azure/service-bus-messaging/message-sessions)

Ou via le code avec la propriété [RequiresSession](https://docs.microsoft.com/en-us/dotnet/api/microsoft.servicebus.messaging.queuedescription.requiressession?view=azure-dotnet)

### Creation des files d'attentes

Dans un premier temps nous allons créer 2 files d'attentes:

- sample.request: sans session, file d'attente pour envoyer les messages de Bob à traiter par Eve
- sample.retry: avec session, file d'attente pour envoyer la réponse à Bob

```c#
static async Task Main()
{
  var ConnectionString = "<your_connection_string>";
  var RequestQueueName = "sample.request";
  var ReplyQueueName = "sample.reply";

  // create queues
  await Task.WhenAll(CreateQueueAsync(connectionString, requestQueueName, false), CreateQueueAsync(connectionString, replyQueueName, true));
}

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

Bob envoie un message avec la propriété ReplyTo égal au chemin d'accès à la file d'attente de réponse et ReplyToSessionId égal à l'identifiant sa session.
Une fois le message envoyé Bob écoute sa session.

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

### Thread du répondeur (Eve)

Eve écoute la file d'attente sur laquelle Bob envoie ses messages.
Une fois qu'un message est intercepté, elle envoie un message sur le chemin d'accès défini par la propriété ReplyTo.
Le message envoyé contient la propriété SessionId défini à partir la propriété ReplyToSessionId du message intercepté (et pour le suivi il est recommandé de définir la propriété CorrelationId avec l'id de message réceptionné).

```c#
public static class SampleThreadFactory
{
...
  public static Thread CreateRequestor(string threadId, string connectionString, string requestQueueName, string replyQueueName)
  {
    return new Thread(async () =>
    {
      var messageSender = new MessageSender(connectionString, requestQueueName);
      var sessionId = Guid.NewGuid().ToString();

      var message = new Message
      {
        MessageId = threadId,
        ReplyTo = new ServiceBusConnectionStringBuilder(connectionString) { EntityPath = replyQueueName }.ToString(),
        ReplyToSessionId = sessionId,
        TimeToLive = TimeSpan.FromMinutes(2)
      };

      Console.WriteLine($"{threadId} send a message to '{requestQueueName}' with replyToSessionId='{message.ReplyToSessionId}' and entityPath='{replyQueueName}'\n");
      await messageSender.SendAsync(message);

      SessionClient sessionClient = new SessionClient(connectionString, replyQueueName);
      var session = await sessionClient.AcceptMessageSessionAsync(sessionId);

      Console.WriteLine($"{threadId}'s waiting a reply message from '{replyQueueName}' with sessionId='{sessionId}'...\n");
      Message sessionMessage = await session.ReceiveAsync();

      Console.WriteLine($"{threadId} received a reply message from '{replyQueueName}' with sessionId='{sessionMessage.SessionId} (original: {sessionId})'\n");
    });
  }
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

![result-single](/blog/images/content/200322-request-response-pattern-with-azure-service-bus/4.png)

Et aucun problème avec plusieurs envoyeur, chacun reçoit le message de réponse qui lui est destiné.

![result-multiple](/blog/images/content/200322-request-response-pattern-with-azure-service-bus/5.png)

**Voilà!**

## Sources

### Repository:

- [https://github.com/Golapadeog/sample-azure-service-bus-request-response](https://github.com/Golapadeog/sample-azure-service-bus-request-response)

### Documentation:

- [https://www.enterpriseintegrationpatterns.com/patterns/messaging/RequestReplyJmsExample.html](https://www.enterpriseintegrationpatterns.com/patterns/messaging/RequestReplyJmsExample.html)
- [https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-amqp-request-response](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-amqp-request-response)
- [https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messages-payloads](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messages-payloads)
- [https://docs.microsoft.com/en-us/azure/service-bus-messaging/message-sessions](https://docs.microsoft.com/en-us/azure/service-bus-messaging/message-sessions)
- [https://en.wikipedia.org/wiki/Request%E2%80%93response](https://en.wikipedia.org/wiki/Request%E2%80%93response)