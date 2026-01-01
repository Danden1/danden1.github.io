---
title: Notification System Design Using Server-Sent Events (SSE)
description: A deep dive into designing a notification system using Server-Sent Events
date: 2026-01-01 21:48:00 +0900
categories: [Backend,Design]
tags: [Backend, SSE, Notification System, Redis]
# mermaid: true
---

I participated in a side project. I developed a notification system using Server-Sent Events (SSE).

The initial implementation was based on SSE and Redis Pub/Sub.  
However, while reviewing the system, I identified several issues with this approach.

## Initial Implementation

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void createNotification(NotificationEvent event) {
    
    //save db
    Notification notification = notificationRepository.save(new Notfication());

    // send redis
    notificationPublisher.publish(...);
}


//sub from redis
//Can't apply retry logic
public class NotificationSubscriber implements MessageListener {
    ...
    public void onMessage(Message message, byte[] pattern) {
        try {
            ...
            sseService.sendToBrowser(notification.getMemberId(), payload);
            ...
        } catch (Exception e) {
            //....
        }
    }
}

//sseService
private final Map<Long, SseEmitter> emitters = new ConcurrentHashMap<>();

public SseEmitter connect(Member member) {
    Long memberId = member.getId();
    SseEmitter emitter = new SseEmitter(30 * 60 * 1000L);
    emitters.put(memberId, emitter);

    emitter.onCompletion(() -> emitters.remove(memberId));
    emitter.onTimeout(() -> emitters.remove(memberId));
    emitter.onError(e -> emitters.remove(memberId));


    try {
        emitter.send(SseEmitter.event().name("ok").data("sse connection"));
    } catch (IOException e) {
        emitter.completeWithError(e);
    }
    return emitter;
}
```

Notifications are critical.
They must be delivered reliably, and duplicate messages must be avoided.

As you know, Redis Pub/Sub broadcasts messages.
When a new subscriber is added, all subscribers receive the same message.

From a logical perspective, the code above works because the connection state is stored in memory.

However, Redis Pub/Sub does not persist messages, which makes it difficult to apply retry logic.
(While it is possible to store the state in a database and re-read messages from the server, this approach raises concerns about duplicate processing and database load.)

When many notifications are made, this make huge traffic.

## How to Improve the Design


### Use Redis Stream

I considered using Redis Streams.

Redis Streams do not broadcast messages.
When consumers belong to the same consumer group, each message is processed by only one consumer.

In addition, if a consumer does not acknowledge a message, it remains in the pending state, which allows retry processing on a scheduled basis.

It is possible to periodically read pending messages and process them.
Because Redis Streams supports min-idle-time, only messages that have been pending for more than a certain number of seconds can be retried.

However, this also introduces a risk.
When a message is claimed by another server to be retried, it enters the pending state again.
If messages in this state are processed incorrectly, duplicate notifications may be delivered.

However, this approach introduces another problem.


### How Should SSE Connections Be Managed?

Since only one consumer (server) can receive a message, the notification must be delivered by the server that owns the SSE connection for that user.

#### SSE Session Manager

To summarize first, this approach seems very difficult and requires further consideration.
In some cases, a broadcast-based approach may actually be more appropriate.

SSE connections themselves cannot be shared across servers.
Therefore, the system must track which server currently owns a user’s SSE connection.

This requires a session manager.

- When a client first establishes an SSE connection, store a (memberId → serverId) mapping in Redis with a TTL (a round-robin strategy or similar approach may be needed).
- The session manager reads Redis to determine which server should deliver the notification, sends the message, and refreshes the TTL.

However, this approach has its own problems.
If a specific server goes down, the session manager cannot deliver notifications to that server.

To prevent new messages from being routed to a downed server, additional mechanisms such as health checks seem to be required.


## Conclusion

While the basic implementation itself is relatively straightforward, designing an SSE-based notification system that properly aligns with business requirements proved to be more complex than expected.

Achieving exactly-once delivery semantics requires careful consideration of many factors, and the design inevitably involves multiple trade-offs.

Even with Redis Streams, several challenges remain.

In particular, guaranteeing message ordering when processing pending messages is non-trivial, and this area requires further refinement.