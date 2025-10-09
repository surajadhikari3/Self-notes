
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


-------------------------------

gotcha—here’s a drop-in Angular setup that:

- hydrates from the backend `/recent` (authoritative),
    
- keeps a rolling cache in `localStorage` so a refresh/tab reopen paints instantly,
    
- stays live via `/stream`,
    
- dedupes and caps to the last N events,
    
- needs no extra clicks after page refresh.
    

---

# 1) app.module.ts

```ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { HttpClientModule } from '@angular/common/http';
import { AppComponent } from './app.component';

@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule, HttpClientModule],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

# 2) environment.ts

```ts
export const environment = {
  production: false,
  apiBase: 'http://localhost:8080/api/kafka'
};
```

# 3) kafka-event.service.ts

```ts
import { Injectable, NgZone } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, firstValueFrom } from 'rxjs';
import { environment } from '../../environments/environment';

@Injectable({ providedIn: 'root' })
export class KafkaEventService {
  private readonly KEY = 'kafka_events_v1';
  private readonly MAX = 500;                 // keep last N
  private eventSource: EventSource | null = null;
  private saving = false;

  private readonly eventsSubject = new BehaviorSubject<string[]>([]);
  public readonly events$ = this.eventsSubject.asObservable();

  constructor(private http: HttpClient, private zone: NgZone) {}

  /** Call once (e.g., AppComponent ngOnInit) */
  async init(): Promise<void> {
    // 1) seed from local cache (instant paint)
    const cached = this.loadCache();
    this.eventsSubject.next(cached);

    // 2) get authoritative backlog from server
    try {
      const recent = await firstValueFrom(
        this.http.get<string[]>(`${environment.apiBase}/recent?limit=${this.MAX}`)
      );
      this.setAndPersist(this.mergeDedupe(cached, recent));
    } catch (e) {
      // backend might be starting; keep cached UI and continue
      console.warn('recent fetch failed, using cache only for now', e);
    }

    // 3) open live stream (SSE auto-reconnects)
    this.openStream();
  }

  private openStream() {
    // Use NgZone to avoid Angular change detection storms
    this.zone.runOutsideAngular(() => {
      this.eventSource = new EventSource(`${environment.apiBase}/stream`);

      this.eventSource.onmessage = (evt: MessageEvent) => {
        const next = this.addEvent(evt.data);
        // jump back into Angular zone to update UI
        this.zone.run(() => this.eventsSubject.next(next));
      };

      this.eventSource.onerror = (err) => {
        console.error('SSE error', err);
        // Let browser auto-reconnect; do not close() here
      };
    });
  }

  /** Current list accessor if needed */
  getSnapshot(): string[] {
    return this.eventsSubject.getValue();
  }

  /** Cleanup (call on app destroy if you wish) */
  close() {
    this.eventSource?.close();
    this.eventSource = null;
  }

  /** ---------- helpers ---------- */

  private addEvent(e: string): string[] {
    const curr = this.eventsSubject.getValue();
    const next = [...curr, e].slice(-this.MAX);
    this.persist(next);
    return next;
  }

  private setAndPersist(list: string[]) {
    const next = list.slice(-this.MAX);
    this.eventsSubject.next(next);
    this.persist(next);
  }

  private mergeDedupe(a: string[], b: string[]): string[] {
    // simple string dedupe; if you later add IDs, dedupe by id
    const seen = new Set<string>();
    const merged: string[] = [];
    for (const v of [...a, ...b]) {
      if (!seen.has(v)) { seen.add(v); merged.push(v); }
    }
    return merged.slice(-this.MAX);
  }

  private loadCache(): string[] {
    try {
      const raw = localStorage.getItem(this.KEY);
      return raw ? JSON.parse(raw) : [];
    } catch { return []; }
  }

  private persist(list: string[]) {
    // throttle writes so we don’t hammer localStorage
    if (this.saving) return;
    this.saving = true;
    queueMicrotask(() => {
      try { localStorage.setItem(this.KEY, JSON.stringify(list)); }
      finally { this.saving = false; }
    });
  }
}
```

# 4) component usage

## entity-list.component.ts (or any component showing events)

```ts
import { Component, OnDestroy, OnInit } from '@angular/core';
import { Subscription } from 'rxjs';
import { KafkaEventService } from '../services/kafka-event.service';

@Component({
  selector: 'app-entity-list',
  templateUrl: './entity-list.component.html'
})
export class EntityListComponent implements OnInit, OnDestroy {
  events$ = this.kafkaSvc.events$;    // async stream for template
  private sub: Subscription | null = null;

  constructor(private kafkaSvc: KafkaEventService) {}

  async ngOnInit() {
    await this.kafkaSvc.init(); // hydrate + connect SSE
    // If you need imperative access, you can also subscribe:
    // this.sub = this.events$.subscribe(list => console.log('events size', list.length));
  }

  ngOnDestroy() {
    this.sub?.unsubscribe();
    // optional: do NOT close SSE here if other components also use it.
    // If this is a global/singleton stream, leave it open.
  }
}
```

## entity-list.component.html

```html
<div class="container">
  <ng-container *ngIf="events$ | async as events">
    <ng-container *ngFor="let e of events">
      <p>{{ e }}</p>
    </ng-container>
  </ng-container>
</div>
```

---

## how this keeps state across refresh/close

- While running, every new event is pushed into a **BehaviorSubject** and **persisted to `localStorage`** (last 500 items).
    
- On a **refresh or tab reopen**, `init()`:
    
    1. paints the cached list immediately (fast UI),
        
    2. fetches `/recent` from your Spring controller to get the authoritative last N,
        
    3. reopens `/stream` for live updates (server also replays the ring buffer at connect).
        
- Because we persist on every event, closing the tab is safe—on next open, you still have a cache to paint instantly, and `/recent` reconciles any gap.
    

You can tune:

- `MAX` (client) and `MAX_RECENT` (server) to your retention needs,
    
- switch to an object shape `{id, ts, payload}` later for perfect dedupe via `id`,
    
- scope the service as a true singleton (providedIn root) so the SSE stream is shared across pages in the same SPA session.