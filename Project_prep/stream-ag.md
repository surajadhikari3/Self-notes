
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