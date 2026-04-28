# gRPC System Design Interview Guide

## Table of Contents
1. [Classic System Design Questions](#classic-system-design-questions)
2. [Microservices Architecture](#microservices-architecture)
3. [Scalability Patterns](#scalability-patterns)
4. [Production Considerations](#production-considerations)
5. [Real-World Scenarios](#real-world-scenarios)

---

## Classic System Design Questions

### Question 1: Design a Microservices-Based Insurance Platform

**Requirements:**
- Handle 10,000 requests/second
- Support multiple services (Prospect, Illustration, Policy, Claims)
- Ensure data consistency
- 99.9% uptime SLA
- Handle peak loads during enrollment periods

**Solution Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                     API Gateway (Kong/NGINX)                 │
│  - Rate limiting (1000 req/min per client)                   │
│  - JWT validation                                            │
│  - SSL termination                                           │
│  - Request routing                                           │
└─────────┬───────────────────────────┬───────────────────────┘
          │                           │
          │ REST (External)           │ gRPC (Internal)
          │                           │
┌─────────▼─────────┐      ┌─────────▼──────────────────────┐
│   BFF Layer       │      │   Service Mesh (Istio)         │
│  (Backend for     │      │  - Service discovery           │
│   Frontend)       │      │  - Load balancing              │
│  - REST → gRPC    │      │  - Circuit breaking            │
│  - Format data    │      │  - Retry policies              │
│    for UI         │      │  - mTLS authentication         │
└───────────────────┘      └────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
           ┌────────▼────────┐ ┌─────▼──────┐ ┌───────▼────────┐
           │ Prospect Service│ │Illustration│ │  Policy Service│
           │   (gRPC)        │ │  Service   │ │    (gRPC)      │
           │                 │ │   (gRPC)   │ │                │
           │ ┌─────────────┐ │ │┌──────────┐│ │ ┌────────────┐ │
           │ │ProspectGrpc │ │ ││Illus Grpc││ │ │Policy Grpc │ │
           │ │   Service   │ │ ││  Service ││ │ │  Service   │ │
           │ └─────────────┘ │ │└──────────┘│ │ └────────────┘ │
           │                 │ │            │ │                │
           │ ┌─────────────┐ │ │┌──────────┐│ │ ┌────────────┐ │
           │ │  Business   │ │ ││ Business ││ │ │  Business  │ │
           │ │   Logic     │ │ ││  Logic   ││ │ │   Logic    │ │
           │ └─────────────┘ │ │└──────────┘│ │ └────────────┘ │
           │                 │ │            │ │                │
           │ ┌─────────────┐ │ │┌──────────┐│ │ ┌────────────┐ │
           │ │ SQL Server  │ │ ││SQL Server││ │ │ SQL Server │ │
           │ │ Database    │ │ ││ Database ││ │ │  Database  │ │
           │ └─────────────┘ │ │└──────────┘│ │ └────────────┘ │
           └─────────────────┘ └────────────┘ └────────────────┘
                    │                 │                 │
                    └─────────────────┼─────────────────┘
                                      │
                          ┌───────────▼───────────┐
                          │  Shared Services      │
                          │  ┌─────────────────┐  │
                          │  │ Event Hub       │  │
                          │  │ (Async Events)  │  │
                          │  └─────────────────┘  │
                          │  ┌─────────────────┐  │
                          │  │ Redis Cache     │  │
                          │  │ (Session/Data)  │  │
                          │  └─────────────────┘  │
                          │  ┌─────────────────┐  │
                          │  │ Application     │  │
                          │  │ Insights        │  │
                          │  │ (Monitoring)    │  │
                          │  └─────────────────┘  │
                          └───────────────────────┘
```

**Key Design Decisions:**

#### 1. Why gRPC for Internal Communication?

**Justification:**
```
REST (JSON/HTTP1.1):
- Payload: ~230 bytes
- Connections: 6 per client (head-of-line blocking)
- Latency: 15ms average
- Throughput: 3,500 req/sec per instance

gRPC (Protobuf/HTTP2):
- Payload: ~85 bytes (63% smaller)
- Connections: 1 per client (multiplexed)
- Latency: 5ms average (3x faster)
- Throughput: 10,000 req/sec per instance (2.8x more)

At 10,000 req/sec:
- REST needs: 3 instances minimum + overhead = 5 instances
- gRPC needs: 1 instance + redundancy = 2 instances
- Cost savings: 60% fewer instances
```

#### 2. Data Consistency Strategy

**Saga Pattern for Distributed Transactions:**

```java
// Example: Create Illustration requires Prospect to exist
public class CreateIllustrationSaga {
    
    public void execute(IllustrationRequest request) {
        // Step 1: Validate prospect exists (gRPC call)
        ProspectResponse prospect = prospectServiceClient.getProspect(
            GetProspectRequest.newBuilder()
                .setProspectId(request.getProspectId())
                .build()
        );
        
        if (prospect.getStatus() != 200) {
            // Compensating action: Fail fast
            throw new ProspectNotFoundException(request.getProspectId());
        }
        
        // Step 2: Create illustration
        IllustrationEntity illustration = createIllustration(request);
        
        try {
            // Step 3: Publish event for downstream services
            eventHub.publish(IllustrationCreatedEvent.builder()
                .illustrationId(illustration.getId())
                .prospectId(request.getProspectId())
                .build()
            );
        } catch (Exception e) {
            // Compensating action: Delete illustration
            deleteIllustration(illustration.getId());
            throw e;
        }
    }
}
```

**Event-Driven Consistency:**
```protobuf
// events.proto
message IllustrationCreatedEvent {
  string illustration_id = 1;
  string prospect_id = 2;
  int64 timestamp = 3;
  string created_by = 4;
}

// Downstream services subscribe to events
service PolicyService {
  // Listens to IllustrationCreatedEvent
  // Creates policy draft when illustration approved
}
```

#### 3. Caching Strategy

**Multi-Layer Cache:**
```java
@Service
public class ProspectServiceImpl {
    
    @Cacheable(value = "prospects", key = "#id")
    public ProspectModel getProspect(String id) {
        // Cache miss: Query database
        return prospectRepository.findById(id)
            .map(this::toModel)
            .orElseThrow();
    }
    
    @CachePut(value = "prospects", key = "#result.id")
    public ProspectModel createProspect(ProspectModel model) {
        ProspectEntity entity = prospectRepository.save(toEntity(model));
        return toModel(entity);
    }
    
    @CacheEvict(value = "prospects", key = "#id")
    public void updateProspect(String id, ProspectModel model) {
        // Update invalidates cache
        prospectRepository.save(toEntity(model));
    }
}
```

**Cache Configuration:**
```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 600000  # 10 minutes
      cache-null-values: false

# Redis Cluster for high availability
redis:
  cluster:
    nodes:
      - redis-node-1:6379
      - redis-node-2:6379
      - redis-node-3:6379
```

#### 4. Load Balancing

**Client-Side Load Balancing with gRPC:**
```java
@Configuration
public class GrpcClientConfig {
    
    @Bean
    public ManagedChannel prospectServiceChannel() {
        return ManagedChannelBuilder
            // DNS-based service discovery (Kubernetes)
            .forTarget("dns:///prospect-service:9090")
            
            // Client-side load balancing
            .defaultLoadBalancingPolicy("round_robin")
            
            // Connection pooling
            .maxInboundMessageSize(10_000_000)
            .keepAliveTime(30, TimeUnit.SECONDS)
            
            // Health checking
            .enableRetry()
            .maxRetryAttempts(3)
            
            .usePlaintext()
            .build();
    }
}
```

**Kubernetes Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: prospect-service
spec:
  type: ClusterIP
  selector:
    app: prospect-service
  ports:
  - port: 9090
    targetPort: grpc
    name: grpc
```

**Horizontal Pod Autoscaler:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: prospect-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: prospect-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

### Question 2: Design a Real-Time Notification System Using gRPC

**Requirements:**
- Send real-time notifications to agents when illustration status changes
- Support 1000+ concurrent connections
- < 100ms latency for notifications
- Handle agent disconnections gracefully

**Solution:**

```protobuf
syntax = "proto3";

service NotificationService {
  // Server-side streaming: One connection, many notifications
  rpc Subscribe(SubscriptionRequest) returns (stream Notification);
  
  // Publish notification (from other services)
  rpc Publish(PublishRequest) returns (PublishResponse);
}

message SubscriptionRequest {
  string agent_id = 1;
  repeated string event_types = 2;  // Filter which events to receive
}

message Notification {
  string notification_id = 1;
  string event_type = 2;
  string title = 3;
  string message = 4;
  int64 timestamp = 5;
  map<string, string> metadata = 6;
}

message PublishRequest {
  string target_agent_id = 1;
  Notification notification = 2;
}

message PublishResponse {
  bool success = 1;
  int32 delivered_count = 2;
}
```

**Implementation:**
```java
@GrpcService
public class NotificationGrpcService 
    extends NotificationServiceGrpc.NotificationServiceImplBase {
    
    // Thread-safe map of agent → StreamObserver
    private final ConcurrentHashMap<String, StreamObserver<Notification>> subscribers = 
        new ConcurrentHashMap<>();
    
    @Override
    public void subscribe(SubscriptionRequest request,
                         StreamObserver<Notification> responseObserver) {
        String agentId = request.getAgentId();
        
        // Store subscriber
        subscribers.put(agentId, responseObserver);
        
        // Send confirmation
        Notification welcome = Notification.newBuilder()
            .setNotificationId(UUID.randomUUID().toString())
            .setEventType("SYSTEM")
            .setTitle("Connected")
            .setMessage("You are now subscribed to notifications")
            .setTimestamp(System.currentTimeMillis())
            .build();
        responseObserver.onNext(welcome);
        
        // Keep connection alive (stream stays open)
        // Notifications will be sent via publish() method
        
        // Cleanup on disconnect
        ServerCallStreamObserver<Notification> callObserver = 
            (ServerCallStreamObserver<Notification>) responseObserver;
        
        callObserver.setOnCancelHandler(() -> {
            subscribers.remove(agentId);
            log.info("Agent {} disconnected", agentId);
        });
    }
    
    @Override
    public void publish(PublishRequest request,
                       StreamObserver<PublishResponse> responseObserver) {
        String targetAgentId = request.getTargetAgentId();
        Notification notification = request.getNotification();
        
        StreamObserver<Notification> subscriber = subscribers.get(targetAgentId);
        
        int deliveredCount = 0;
        boolean success = false;
        
        if (subscriber != null) {
            try {
                subscriber.onNext(notification);
                deliveredCount = 1;
                success = true;
            } catch (Exception e) {
                // Agent disconnected, remove from map
                subscribers.remove(targetAgentId);
                log.error("Failed to deliver to agent {}", targetAgentId, e);
            }
        } else {
            // Agent not connected, could store for later delivery
            log.warn("Agent {} not subscribed", targetAgentId);
        }
        
        PublishResponse response = PublishResponse.newBuilder()
            .setSuccess(success)
            .setDeliveredCount(deliveredCount)
            .build();
        
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
    
    // Broadcast to all agents
    public void broadcastNotification(Notification notification) {
        int delivered = 0;
        List<String> failed = new ArrayList<>();
        
        for (Map.Entry<String, StreamObserver<Notification>> entry : 
             subscribers.entrySet()) {
            try {
                entry.getValue().onNext(notification);
                delivered++;
            } catch (Exception e) {
                failed.add(entry.getKey());
            }
        }
        
        // Cleanup failed connections
        failed.forEach(subscribers::remove);
        
        log.info("Broadcast notification to {} agents, {} failed", 
            delivered, failed.size());
    }
}
```

**Client (JavaScript/Web):**
```javascript
// Using grpc-web
const client = new NotificationServiceClient('https://api.example.com');

const request = new SubscriptionRequest();
request.setAgentId('AGENT123');
request.setEventTypesList(['ILLUSTRATION_APPROVED', 'NEW_APPLICATION']);

const stream = client.subscribe(request, {});

stream.on('data', (notification) => {
  console.log('Received notification:', notification.getTitle());
  showToast(notification.getMessage());
});

stream.on('error', (err) => {
  console.error('Stream error:', err);
  // Reconnect with exponential backoff
  setTimeout(() => reconnect(), 5000);
});

stream.on('end', () => {
  console.log('Stream ended');
});
```

**Integration with Event System:**
```java
@Service
public class EventHubListener {
    
    private final NotificationGrpcService notificationService;
    
    @EventHubConsumer(topic = "illustration-events")
    public void handleIllustrationEvent(IllustrationEvent event) {
        if (event.getType() == EventType.APPROVED) {
            Notification notification = Notification.newBuilder()
                .setNotificationId(UUID.randomUUID().toString())
                .setEventType("ILLUSTRATION_APPROVED")
                .setTitle("Illustration Approved")
                .setMessage(String.format(
                    "Illustration %s has been approved", 
                    event.getIllustrationId()))
                .setTimestamp(System.currentTimeMillis())
                .putMetadata("illustrationId", event.getIllustrationId())
                .build();
            
            // Send to specific agent
            notificationService.publish(
                PublishRequest.newBuilder()
                    .setTargetAgentId(event.getAgentId())
                    .setNotification(notification)
                    .build(),
                new StreamObserver<>() {
                    // Handle response
                }
            );
        }
    }
}
```

**Scalability Considerations:**

1. **Connection Limits:**
   - 1000 concurrent streams per server instance
   - Use multiple instances behind load balancer
   - Sticky sessions to maintain connection to same instance

2. **Memory Management:**
   ```java
   // Monitor subscriber count
   @Scheduled(fixedRate = 60000)  // Every minute
   public void checkSubscribers() {
       int count = subscribers.size();
       metrics.gauge("notification.subscribers", count);
       
       if (count > 800) {
           log.warn("High subscriber count: {}", count);
           // Alert ops team to scale up
       }
   }
   ```

3. **Persistence for Offline Agents:**
   ```java
   @Override
   public void publish(PublishRequest request, ...) {
       if (!subscribers.containsKey(targetAgentId)) {
           // Store in database for later delivery
           notificationRepository.save(
               OfflineNotification.builder()
                   .agentId(targetAgentId)
                   .notification(serializeNotification(request.getNotification()))
                   .createdAt(Instant.now())
                   .build()
           );
       }
   }
   
   // On reconnect, deliver pending notifications
   @Override
   public void subscribe(SubscriptionRequest request, ...) {
       String agentId = request.getAgentId();
       
       // Deliver pending notifications
       List<OfflineNotification> pending = 
           notificationRepository.findByAgentIdAndDeliveredFalse(agentId);
       
       for (OfflineNotification offline : pending) {
           responseObserver.onNext(
               deserializeNotification(offline.getNotification())
           );
           offline.setDelivered(true);
       }
       
       notificationRepository.saveAll(pending);
   }
   ```

---

## Scalability Patterns

### 1. Database Sharding

**Problem:** Single database becomes bottleneck at high scale

**Solution: Shard by Prospect ID**

```java
@Configuration
public class ShardingConfig {
    
    private static final int NUM_SHARDS = 4;
    
    public DataSource getShardDataSource(String prospectId) {
        int shard = Math.abs(prospectId.hashCode()) % NUM_SHARDS;
        
        return switch (shard) {
            case 0 -> dataSource0;
            case 1 -> dataSource1;
            case 2 -> dataSource2;
            case 3 -> dataSource3;
            default -> throw new IllegalStateException();
        };
    }
}

@Service
public class ProspectServiceImpl {
    
    private final ShardingConfig shardingConfig;
    
    public ProspectModel getProspect(String prospectId) {
        DataSource dataSource = shardingConfig.getShardDataSource(prospectId);
        
        // Use shard-specific repository
        JpaRepository<ProspectEntity, String> repo = 
            new ProspectRepository(dataSource);
        
        return repo.findById(prospectId)
            .map(this::toModel)
            .orElseThrow();
    }
}
```

**Proto for Cross-Shard Queries:**
```protobuf
service ProspectService {
  // Single prospect (hits one shard)
  rpc GetProspect(GetProspectRequest) returns (GetProspectResponse);
  
  // Multiple prospects (scatter-gather across shards)
  rpc BatchGetProspects(BatchGetProspectsRequest) 
      returns (stream ProspectResponse);
}

message BatchGetProspectsRequest {
  repeated string prospect_ids = 1;
}
```

```java
@Override
public void batchGetProspects(BatchGetProspectsRequest request,
                              StreamObserver<ProspectResponse> responseObserver) {
    List<CompletableFuture<ProspectModel>> futures = new ArrayList<>();
    
    for (String prospectId : request.getProspectIdsList()) {
        // Query each shard in parallel
        CompletableFuture<ProspectModel> future = 
            CompletableFuture.supplyAsync(() -> getProspect(prospectId));
        futures.add(future);
    }
    
    // Stream results as they complete
    for (CompletableFuture<ProspectModel> future : futures) {
        future.thenAccept(prospect -> {
            ProspectResponse response = ProspectResponse.newBuilder()
                .setProspect(toProto(prospect))
                .build();
            responseObserver.onNext(response);
        });
    }
    
    // Wait for all to complete
    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
        .thenRun(() -> responseObserver.onCompleted());
}
```

### 2. Rate Limiting

**Token Bucket Algorithm:**

```java
@Component
public class RateLimitInterceptor implements ServerInterceptor {
    
    private final LoadingCache<String, Bucket> buckets = CacheBuilder.newBuilder()
        .expireAfterWrite(1, TimeUnit.HOURS)
        .build(new CacheLoader<String, Bucket>() {
            public Bucket load(String key) {
                // 100 requests per minute per client
                Refill refill = Refill.intervally(100, Duration.ofMinutes(1));
                Bandwidth limit = Bandwidth.classic(100, refill);
                return Bucket.builder()
                    .addLimit(limit)
                    .build();
            }
        });
    
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
        ServerCall<ReqT, RespT> call,
        Metadata headers,
        ServerCallHandler<ReqT, RespT> next) {
        
        String clientId = extractClientId(headers);
        Bucket bucket = buckets.get(clientId);
        
        if (bucket.tryConsume(1)) {
            // Allow request
            return next.startCall(call, headers);
        } else {
            // Rate limit exceeded
            call.close(
                Status.RESOURCE_EXHAUSTED
                    .withDescription("Rate limit exceeded: 100 req/min"),
                new Metadata()
            );
            return new ServerCall.Listener<>() {};
        }
    }
}
```

**Distributed Rate Limiting with Redis:**

```java
@Component
public class RedisRateLimiter {
    
    private final RedisTemplate<String, String> redis;
    
    public boolean allowRequest(String clientId) {
        String key = "rate_limit:" + clientId;
        long nowSeconds = Instant.now().getEpochSecond();
        long windowStart = nowSeconds - 60;  // 1 minute window
        
        // Remove old entries
        redis.opsForZSet().removeRangeByScore(key, 0, windowStart);
        
        // Count requests in current window
        Long count = redis.opsForZSet().count(key, windowStart, nowSeconds);
        
        if (count != null && count >= 100) {
            return false;  // Rate limit exceeded
        }
        
        // Add current request
        redis.opsForZSet().add(key, UUID.randomUUID().toString(), nowSeconds);
        redis.expire(key, Duration.ofMinutes(2));
        
        return true;
    }
}
```

### 3. Circuit Breaker

**Resilience4j Integration:**

```java
@Configuration
public class CircuitBreakerConfig {
    
    @Bean
    public CircuitBreakerRegistry circuitBreakerRegistry() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .slidingWindowSize(100)
            .failureRateThreshold(50)  // Open at 50% failure
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .permittedNumberOfCallsInHalfOpenState(10)
            .build();
        
        return CircuitBreakerRegistry.of(config);
    }
}

@Service
public class ResilientProspectClient {
    
    private final CircuitBreaker circuitBreaker;
    private final ProspectServiceBlockingStub stub;
    
    public ProspectResponse getProspect(String prospectId) {
        return circuitBreaker.executeSupplier(() -> {
            GetProspectRequest request = GetProspectRequest.newBuilder()
                .setProspectId(prospectId)
                .build();
            
            return stub.getProspect(request);
        });
    }
}
```

**Circuit Breaker Events:**
```java
circuitBreaker.getEventPublisher()
    .onStateTransition(event -> {
        log.warn("Circuit breaker state changed: {} -> {}", 
            event.getStateTransition().getFromState(),
            event.getStateTransition().getToState());
        
        // Alert monitoring system
        if (event.getStateTransition().getToState() == CircuitBreaker.State.OPEN) {
            alertService.sendAlert(
                "Circuit breaker opened for ProspectService",
                Severity.HIGH
            );
        }
    });
```

---

## Production Considerations

### 1. Monitoring & Observability

**Metrics Collection:**
```java
@GrpcService(interceptors = MetricsInterceptor.class)
public class ProspectGrpcService extends ... {
    
    private final MeterRegistry meterRegistry;
    
    @Override
    public void createAndSaveProspect(...) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            // Process request
            prospectService.createAndSaveProspect(model);
            
            // Record success
            meterRegistry.counter("grpc.requests.success",
                "method", "createAndSaveProspect",
                "service", "ProspectService"
            ).increment();
            
        } catch (Exception e) {
            // Record failure
            meterRegistry.counter("grpc.requests.failure",
                "method", "createAndSaveProspect",
                "service", "ProspectService",
                "error", e.getClass().getSimpleName()
            ).increment();
            
            throw e;
        } finally {
            sample.stop(meterRegistry.timer("grpc.request.duration",
                "method", "createAndSaveProspect"
            ));
        }
    }
}
```

**Distributed Tracing:**
```java
@Component
public class TracingInterceptor implements ServerInterceptor {
    
    private final Tracer tracer;
    
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(...) {
        // Extract trace context from metadata
        String traceId = headers.get(
            Metadata.Key.of("x-trace-id", Metadata.ASCII_STRING_MARSHALLER)
        );
        
        Span span = tracer.nextSpan()
            .name(call.getMethodDescriptor().getFullMethodName())
            .tag("grpc.method", call.getMethodDescriptor().getBareMethodName())
            .tag("trace.id", traceId)
            .start();
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            return next.startCall(call, headers);
        } finally {
            span.finish();
        }
    }
}
```

**Dashboard Queries (Azure Application Insights):**
```kusto
// Request rate per minute
requests
| where cloud_RoleName == "uaa-middleware"
| where name contains "grpc"
| summarize RequestCount = count() by bin(timestamp, 1m), name
| render timechart

// 95th percentile latency
requests
| where cloud_RoleName == "uaa-middleware"
| where name == "createAndSaveProspect"
| summarize percentiles(duration, 50, 95, 99) by bin(timestamp, 5m)
| render timechart

// Error rate
requests
| where cloud_RoleName == "uaa-middleware"
| where success == false
| summarize ErrorRate = count() by resultCode, name
| order by ErrorRate desc
```

### 2. Security in Production

**Full TLS Setup:**
```java
@Configuration
public class GrpcSecurityConfig {
    
    @Bean
    public GrpcServerBuilderConfigurer grpcServerBuilderConfigurer() 
        throws SSLException {
        
        SslContextBuilder sslContextBuilder = SslContextBuilder.forServer(
            new File("/etc/certs/server.crt"),
            new File("/etc/certs/server.key")
        );
        
        // Mutual TLS: Require client certificates
        sslContextBuilder.trustManager(new File("/etc/certs/ca.crt"));
        sslContextBuilder.clientAuth(ClientAuth.REQUIRE);
        
        SslContext sslContext = sslContextBuilder.build();
        
        return serverBuilder -> {
            serverBuilder.useTransportSecurity(sslContext);
        };
    }
}
```

**JWT Validation:**
```java
@Component
public class JwtAuthInterceptor implements ServerInterceptor {
    
    private final JWTVerifier verifier = JWT.require(
        Algorithm.HMAC256(System.getenv("JWT_SECRET"))
    ).build();
    
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(...) {
        String authHeader = headers.get(
            Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER)
        );
        
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            call.close(Status.UNAUTHENTICATED, new Metadata());
            return new ServerCall.Listener<>() {};
        }
        
        try {
            DecodedJWT jwt = verifier.verify(authHeader.substring(7));
            
            // Add user context for authorization
            Context context = Context.current()
                .withValue(USER_ID, jwt.getClaim("sub").asString())
                .withValue(USER_ROLE, jwt.getClaim("role").asString());
            
            return Contexts.interceptCall(context, call, headers, next);
            
        } catch (JWTVerificationException e) {
            call.close(Status.UNAUTHENTICATED
                .withDescription("Invalid JWT"), new Metadata());
            return new ServerCall.Listener<>() {};
        }
    }
    
    public static final Context.Key<String> USER_ID = Context.key("userId");
    public static final Context.Key<String> USER_ROLE = Context.key("userRole");
}
```

---

## Interview Discussion Points

### Trade-offs Discussion

**Interviewer: "Why not just use REST for everything?"**

**Your Answer:**
"I chose gRPC for internal microservice communication because:

**Advantages:**
1. **Performance**: 3x lower latency (5ms vs 15ms) and 63% smaller payloads
2. **Type Safety**: Protocol Buffers catch errors at compile time, not runtime
3. **Streaming**: Built-in support for real-time features without WebSockets
4. **HTTP/2**: Multiplexing reduces connection overhead

**Trade-offs:**
1. **Browser Support**: gRPC-web needed for browser clients (added complexity)
2. **Debugging**: Binary format harder to inspect than JSON (tooling: grpcurl, Postman)
3. **Learning Curve**: Team needs to learn protobuf syntax and StreamObserver pattern

**Decision**: We use gRPC for internal services (high performance) and REST for external APIs (better compatibility). This hybrid approach gives us the best of both worlds."

---

**Interviewer: "How do you handle a service being down?"**

**Your Answer:**
"Multi-layer resilience strategy:

1. **Circuit Breaker**: Stop calling unhealthy service after 50% failure rate
2. **Retry with Backoff**: 3 retries with exponential backoff (100ms, 200ms, 400ms)
3. **Timeout**: 5-second deadline to prevent indefinite waits
4. **Fallback**: Return cached data or degraded response
5. **Health Checks**: Kubernetes probes remove unhealthy pods from load balancer

Example code:
```java
CircuitBreaker cb = CircuitBreaker.ofDefaults("prospectService");
Retry retry = Retry.of("prospectService", RetryConfig.custom()
    .maxAttempts(3)
    .waitDuration(Duration.ofMillis(100))
    .build());

Supplier<ProspectResponse> supplier = CircuitBreaker
    .decorateSupplier(cb, () -> stub.getProspect(request));

supplier = Retry.decorateSupplier(retry, supplier);

try {
    return supplier.get();
} catch (Exception e) {
    return getCachedProspect(prospectId);  // Fallback
}
```"

---

**Document Purpose:** System design interview preparation for gRPC-based systems  
**Target Audience:** Senior SWE interviews (L4/L5)  
**Estimated Study Time:** 6-8 hours to master all concepts

