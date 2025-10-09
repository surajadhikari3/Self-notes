
You’re _almost_ there. Two things are causing the 503 and the “emitters = 0”:

1. Your SSE endpoint isn’t announcing itself as an SSE stream (so the browser tears it down and Spring returns 503 before any client is registered).
    
2. You only send to emitters when the Kafka value is a `LinkedHashMap`. If your deserializer isn’t the JSON one, the value will be a `String` and you skip sending—so Angular never sees anything.
    

Here’s a tight, working setup.

# 1) Spring Boot – SSE endpoint (fix 503 + keep-alive)

```java
@RestController
@RequestMapping("/api/kafka")
@RequiredArgsConstructor
@CrossOrigin(origins = "*")
public class KafkaController {

  private final KafkaListenerService kafkaListenerService;

  // Tell Spring/Tomcat this is a server-sent event stream
  @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
  public SseEmitter streamEvents() {
    return kafkaListenerService.streamEvents();
  }
}
```

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class KafkaListenerService {

  // thread-safe for concurrent iteration
  private final CopyOnWriteArrayList<SseEmitter> emitters = new CopyOnWriteArrayList<>();

  // Never timeout (or choose a large value like 30 minutes)
  public SseEmitter streamEvents() {
    SseEmitter emitter = new SseEmitter(0L);
    emitters.add(emitter);

    emitter.onCompletion(() -> emitters.remove(emitter));
    emitter.onTimeout(() -> emitters.remove(emitter));
    emitter.onError(e -> {
      log.warn("SSE emitter error: {}", e.getMessage());
      emitters.remove(emitter);
    });

    try {
      // send a first 'hello' so the connection is established and Angular knows it’s open
      emitter.send(SseEmitter.event().name("init").data("connected"));
    } catch (Exception e) {
      emitters.remove(emitter);
      emitter.completeWithError(e);
    }
    return emitter;
  }

  @KafkaListener(
      topics = "${spring.kafka.consumer.properties.topic}",
      containerFactory = "kafkaListenerContainerFactory" // if you’ve customized it
  )
  public void onMessage(ConsumerRecord<String, Object> record, Acknowledgment ack) {
    try {
      if (record.value() == null) {
        log.warn("Null value for key {}", record.key());
        ack.acknowledge();
        return;
      }

      // Normalize payload to String so Angular can display it
      String payload;
      if (record.value() instanceof String s) {
        payload = s;
      } else if (record.value() instanceof Map<?, ?> m) {
        payload = new ObjectMapper().writeValueAsString(m); // or inject/reuse a single ObjectMapper
      } else {
        payload = String.valueOf(record.value());
      }

      // fan-out
      for (SseEmitter emitter : emitters) {
        try {
          emitter.send(SseEmitter.event().name("message").data(payload));
        } catch (Exception e) {
          // drop dead emitters
          emitters.remove(emitter);
          log.warn("Removing dead emitter: {}", e.getMessage());
        }
      }

      ack.acknowledge();
    } catch (Exception e) {
      log.error("Error handling Kafka message key={}, value={}, ex={}",
          record.key(), record.value(), e.getMessage(), e);
    }
  }
}
```

**application.properties (important)**

```
# Do not time out async requests (prevents 503 on long-lived SSE)
spring.mvc.async.request-timeout=0
# If you use manual acks:
spring.kafka.listener.ack-mode=MANUAL_IMMEDIATE
```

> If you expect JSON values from Kafka, make sure your consumer uses the JSON deserializer (or Confluent Avro/JSON as appropriate). Otherwise your values will be plain strings.

# 2) Angular – EventSource usage (works once SSE is correct)

Service:

```ts
@Injectable({ providedIn: 'root' })
export class KafkaEventService {
  private eventSource: EventSource | null = null;

  connectToKafkaStream(): Observable<string> {
    return new Observable<string>((observer) => {
      this.eventSource = new EventSource('http://localhost:8080/api/kafka/stream');

      this.eventSource.onmessage = (event) => {
        observer.next(event.data);        // receive payload string
      };

      this.eventSource.onerror = (err) => {
        observer.error(err);
        this.eventSource?.close();
      };

      return () => this.eventSource?.close();
    });
  }
}
```

Component (your code is fine):

```ts
this.eventSubscription = this.kafkaService.connectToKafkaStream().subscribe({
  next: (event: string) => {
    console.log('Kafka Event Received:', event);
    this.events.push(event);
  },
  error: (err) => console.error('Error receiving Kafka events:', err),
});
```

Template (your code is fine):

```html
<ng-container *ngFor="let event of events">
  <p>{{ event }}</p>
</ng-container>
```

# Quick checklist

-  `/stream` has `produces = MediaType.TEXT_EVENT_STREAM_VALUE`.
    
-  `spring.mvc.async.request-timeout=0` is set.
    
-  On connect, you `send()` an initial event (e.g., `init`) so the browser keeps the stream open.
    
-  In the listener, always convert the payload to a string and send; don’t gate on `LinkedHashMap` only.
    
-  If you use manual acks, set `ack-mode=MANUAL_IMMEDIATE`.
    
-  CORS is enabled (you already have `@CrossOrigin("*")`).
    

After these tweaks, Angular’s first load should return HTTP 200 and your emitter list will contain at least one emitter; when Kafka messages arrive, they’ll be fanned out to the browser immediately.



----------------------------------


Yep—SSE is “live only.” On a browser refresh you lose in-memory state unless you (a) replay from the server or (b) hydrate from client storage. Do both for a great UX:

## A) Server-side replay (authoritative)

Keep a small ring buffer of the last N events; send that backlog to any new SSE subscriber before streaming live updates.

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class KafkaListenerService {

  private final CopyOnWriteArrayList<SseEmitter> emitters = new CopyOnWriteArrayList<>();

  // ring buffer
  private static final int MAX_RECENT = 500;
  private final Deque<String> recent = new ConcurrentLinkedDeque<>();

  public SseEmitter streamEvents() {
    SseEmitter emitter = new SseEmitter(0L);
    emitters.add(emitter);

    emitter.onCompletion(() -> emitters.remove(emitter));
    emitter.onTimeout(() -> emitters.remove(emitter));
    emitter.onError(e -> { emitters.remove(emitter); });

    try {
      // 1) tell client we're on
      emitter.send(SseEmitter.event().name("init").data("connected"));

      // 2) REPLAY recent events
      for (String msg : recent) {
        emitter.send(SseEmitter.event().name("replay").data(msg));
      }
    } catch (Exception e) {
      emitters.remove(emitter);
      emitter.completeWithError(e);
    }
    return emitter;
  }

  @KafkaListener(topics = "${spring.kafka.consumer.properties.topic}")
  public void onMessage(ConsumerRecord<String, Object> record, Acknowledgment ack) {
    try {
      String payload = (record.value() instanceof String s)
          ? s
          : new ObjectMapper().writeValueAsString(record.value());

      // push into ring buffer
      recent.addLast(payload);
      while (recent.size() > MAX_RECENT) recent.pollFirst();

      // fan out
      for (SseEmitter emitter : emitters) {
        try {
          emitter.send(SseEmitter.event().name("message").data(payload));
        } catch (Exception e) {
          emitters.remove(emitter);
        }
      }
      ack.acknowledge();

    } catch (Exception e) {
      log.error("Kafka handle error", e);
    }
  }

  // Optional REST to fetch recent manually
  public List<String> getRecent(int limit) {
    return recent.stream().skip(Math.max(0, recent.size() - limit)).toList();
  }
}
```

Controller (optional REST):

```java
@GetMapping("/recent")
public List<String> recent(@RequestParam(defaultValue = "100") int limit) {
  return kafkaListenerService.getRecent(limit);
}
```

## B) Client-side hydration (fast UX)

Seed the UI from localStorage, then connect to SSE; keep caching as new events arrive so a refresh is instant.

```ts
@Injectable({ providedIn: 'root' })
export class KafkaEventService {
  private readonly KEY = 'kafka_events_v1';
  private eventSource: EventSource | null = null;

  loadCached(): string[] {
    const raw = localStorage.getItem(this.KEY);
    return raw ? JSON.parse(raw) : [];
  }
  saveCached(events: string[]) {
    localStorage.setItem(this.KEY, JSON.stringify(events.slice(-500)));
  }

  connect(): Observable<string> {
    return new Observable<string>((observer) => {
      this.eventSource = new EventSource('http://localhost:8080/api/kafka/stream');

      this.eventSource.onmessage = (e) => observer.next(e.data);
      this.eventSource.onerror = (err) => { observer.error(err); this.eventSource?.close(); };

      return () => this.eventSource?.close();
    });
  }
}
```

Component:

```ts
events: string[] = [];

ngOnInit() {
  // 1) hydrate from cache
  this.events = this.kafkaService.loadCached();

  // 2) (optional) pull server backlog if you exposed /recent
  // this.http.get<string[]>('/api/kafka/recent?limit=200').subscribe(list => {
  //   this.events = list;
  //   this.kafkaService.saveCached(this.events);
  // });

  // 3) live stream
  this.eventSubscription = this.kafkaService.connect().subscribe({
    next: (ev) => {
      this.events.push(ev);
      this.kafkaService.saveCached(this.events);
    },
    error: (err) => console.error(err)
  });
}
```

**Result:**

- New tab/refresh immediately shows cached events.
    
- Server also replays the last N events so every client can catch up accurately.
    
- Live messages continue as before.

---------------

here’s a clean controller you can drop in:

```java
package com.tdsecurities.bishowcase2025.controller;

import com.tdsecurities.bishowcase2025.service.KafkaListenerService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.util.List;

@RestController
@RequestMapping("/api/kafka")
@RequiredArgsConstructor
@CrossOrigin(origins = "*") // adjust for your domains in prod
public class KafkaController {

    private final KafkaListenerService kafkaListenerService;

    // SSE stream endpoint (Angular EventSource connects here)
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter streamEvents() {
        return kafkaListenerService.streamEvents();
    }

    // Optional: fetch recent (server-side replay) to hydrate UI on refresh
    @GetMapping("/recent")
    public List<String> recent(@RequestParam(defaultValue = "100") int limit) {
        return kafkaListenerService.getRecent(limit);
    }

    // Optional: simple health check (useful for debugging 503 vs 200)
    @GetMapping("/health")
    public String health() {
        return "OK";
    }
}
```

If you also want a manual way to clear the buffer during testing:

```java
    @DeleteMapping("/recent")
    public void clearRecent() { kafkaListenerService.clearRecent(); }
```

(Then add `public void clearRecent(){ recent.clear(); }` to the service.)