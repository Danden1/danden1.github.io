---
title: Notification System Design Using Server-Sent Events (SSE)
description: A deep dive into designing a notification system using Server-Sent Events
date: 2026-01-01 21:48:00 +0900
categories: [Backend,Design]
tags: [Backend, SSE, Notification System, Redis]
# mermaid: true
---

In a side project, I developed a notification system using Server-Sent Events (SSE).

The initial implementation was based on SSE and Redis Pub/Sub.  
However, while reviewing the system, I identified several issues with this approach.

## Initial Implementation

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void createNotification(NotificationEvent event) {
    
    // save to database
    Notification notification = notificationRepository.save(new Notification());

    // publish to Redis
    notificationPublisher.publish(...);
}


// Subscribe to Redis
// Retry logic cannot be applied
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

// SSE service
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

Reliable notification delivery was important for this project. Messages needed to arrive without being delivered twice.

Redis Pub/Sub broadcasts each message to every subscriber on the channel.

The code above works because each server keeps its connection state in memory.

However, Redis Pub/Sub does not persist messages, which makes it difficult to apply retry logic.
(While it is possible to store the state in a database and re-read messages from the server, this approach raises concerns about duplicate processing and database load.)

When many notifications are generated, this can create a large amount of network traffic.

## How to Improve the Design


### Use Redis Streams

I considered using Redis Streams.

Redis Streams do not broadcast messages.
When consumers belong to the same consumer group, each message is processed by only one consumer.

In addition, if a consumer does not acknowledge a message, it remains in the pending state, which allows retry processing on a scheduled basis.

It is possible to periodically read pending messages and process them.
Because Redis Streams support `min-idle-time`, only messages that have been pending for more than a certain number of seconds can be retried.

However, this also introduces a risk.
When another server claims a message for retry, the message returns to the pending state. If the retry is not handled carefully, the notification may be delivered twice.

### How Should SSE Connections Be Managed?

Since only one consumer (server) can receive a message, the notification must be delivered by the server that owns the SSE connection for that user.

#### SSE Session Manager

In short, this approach adds a lot of complexity. In some cases, broadcasting the message may be a better choice.

SSE connections themselves cannot be shared across servers.
Therefore, the system must track which server currently owns a user’s SSE connection.

This requires a session manager.

- When a client first establishes an SSE connection, store a (memberId → serverId) mapping in Redis with a TTL (a round-robin strategy or similar approach may be needed).
- The session manager reads Redis to determine which server should deliver the notification, sends the message, and refreshes the TTL.

However, this approach has its own problems.
If a specific server goes down, the session manager cannot deliver notifications to that server.

Health checks are also needed to prevent new messages from being routed to an unavailable server.


## Conclusion

The first implementation was simple, but matching an SSE-based notification system to the business requirements was harder than I expected.

Achieving exactly-once delivery semantics requires careful consideration of many factors, and the design inevitably involves multiple trade-offs.

Even with Redis Streams, several challenges remain.

In particular, guaranteeing message ordering when processing pending messages is non-trivial, and this area requires further refinement.
