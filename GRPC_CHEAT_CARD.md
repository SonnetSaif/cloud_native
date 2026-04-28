# gRPC Interview Cheat Card (Print This!)

## 🎯 30-Second Elevator Pitch
"I implemented a gRPC microservice with Protocol Buffers 3 and Spring Boot 4.0 for an insurance platform, achieving 3x performance improvement over REST. Built 2 services with 59 fields, 100% test coverage, and automated code generation via Gradle protobuf plugin."

## 📊 Key Metrics to Memorize
| Metric | Value | vs REST |
|--------|-------|---------|
| **Payload Size** | 85 bytes | 63% smaller |
| **Latency** | 5ms | 3x faster |
| **Throughput** | 10K req/s | 2.8x more |
| **Services** | 2 (Prospect, Illustration) | - |
| **Fields** | 59 total | - |
| **Test Coverage** | 100% (10 tests) | - |

## 🔧 Tech Stack
- gRPC: 1.59.0
- Protobuf: 3.25.1 (proto3)
- Spring Boot: 4.0.0
- Java: 21
- Testing: JUnit 5 + Mockito

## 📝 Proto Syntax
```protobuf
syntax = "proto3";
service MyService {
  rpc Method(Request) returns (Response);
}
message Request {
  string field1 = 1;         // Field number
  optional string field2 = 2; // Optional
  repeated string field3 = 3; // List
}
```

## ⚙️ Service Implementation
```java
@GrpcService
public class MyService extends MyServiceGrpc.MyServiceImplBase {
  @Override
  public void method(Request req, StreamObserver<Response> resp) {
    try {
      Response r = Response.newBuilder()
        .setField("value").build();
      resp.onNext(r);
      resp.onCompleted(); // ⚠️ MUST call!
    } catch (Exception e) {
      resp.onError(Status.INTERNAL
        .withCause(e).asRuntimeException());
    }
  }
}
```

## 🧪 Testing Pattern
```java
@Mock StreamObserver<Response> responseObserver;
@Test void test() {
  service.method(request, responseObserver);
  ArgumentCaptor<Response> captor = 
    ArgumentCaptor.forClass(Response.class);
  verify(responseObserver).onNext(captor.capture());
  verify(responseObserver).onCompleted();
  assertEquals("expected", captor.getValue().getField());
}
```

## 🚨 StreamObserver Rules
| Method | Usage | Count |
|--------|-------|-------|
| `onNext(value)` | Send response | 1+ times |
| `onCompleted()` | Success end | Exactly 1 |
| `onError(throwable)` | Error end | Exactly 1 |

**CRITICAL:** Always call `onCompleted()` OR `onError()` (never both, never neither!)

## 🔄 RPC Types
```protobuf
rpc Unary(Request) returns (Response);              // Simple call
rpc ServerStream(Request) returns (stream Response); // Download
rpc ClientStream(stream Request) returns (Response); // Upload
rpc BiDi(stream Request) returns (stream Response);  // Chat
```

## ⚡ Performance Why?
1. **Binary**: Protobuf vs JSON (smaller, faster)
2. **HTTP/2**: Multiplexing (1 connection vs 6)
3. **Type-Safe**: Compile-time errors (not runtime)
4. **Streaming**: Built-in (no WebSocket needed)

## 🛠️ Build Config
```gradle
plugins { id 'com.google.protobuf' version '0.9.5' }
dependencies {
  implementation 'net.devh:grpc-spring-boot-starter:3.1.0.RELEASE'
  implementation 'io.grpc:grpc-protobuf:1.59.0'
  implementation 'io.grpc:grpc-stub:1.59.0'
}
protobuf {
  protoc { artifact = 'com.google.protobuf:protoc:3.25.1' }
  plugins { grpc { artifact = 'io.grpc:protoc-gen-grpc-java:1.59.0' }}
  generateProtoTasks { all()*.plugins { grpc {} }}
}
```

## 🚫 Common Errors
| Error | Fix |
|-------|-----|
| `UNIMPLEMENTED` | Override method with `@Override` |
| `UNAVAILABLE` | Check server running on correct port |
| `DEADLINE_EXCEEDED` | Increase timeout: `stub.withDeadlineAfter(10, SECONDS)` |
| Memory leak | Always call `onCompleted()` |

## 🎨 System Design Pattern
```
API Gateway → BFF Layer → gRPC Services → Databases
     ↓            ↓             ↓
  REST API    REST→gRPC    Internal gRPC
  (External)  (Adapter)   (High-perf)
```

## 🔐 Security Layers
1. **TLS**: Transport encryption
2. **JWT**: Authentication (in metadata)
3. **Interceptors**: Authorization
4. **mTLS**: Mutual auth (production)

## 📈 Scalability
- **Load Balance**: DNS round-robin + client-side LB
- **Circuit Breaker**: Stop calling at 50% failure rate
- **Retry**: 3 attempts with exponential backoff
- **Rate Limit**: 100 req/min per client (token bucket)
- **Cache**: Redis for frequent reads (10min TTL)

## 🎯 Interview Questions Lightning Round

**Q: gRPC vs REST?**  
A: gRPC = binary, HTTP/2, type-safe, streaming. REST = JSON, HTTP/1.1, readable, compatible. Use gRPC internally, REST externally.

**Q: How does StreamObserver work?**  
A: Async callback interface. Call `onNext()` for each response, then exactly one of `onCompleted()` (success) or `onError()` (failure).

**Q: What's protobuf versioning?**  
A: Field numbers enable backward compatibility. Can add new optional fields, never remove or change existing numbers.

**Q: How to handle failures?**  
A: Circuit breaker (stop at 50% fail), retry with backoff (3x), timeout (5s), fallback to cache.

**Q: Testing strategy?**  
A: Mock StreamObserver, use ArgumentCaptor, verify onNext/onCompleted called, test success/failure/exception paths.

## 📋 My Project Specifics
**ProspectService:**
- Fields: firstName, lastName, middleName, gender, DOB, age, gsspProspectId, email, contactNo, createdBy, createdAt
- Method: `CreateAndSaveProspect`
- Returns: 201 (Created), 200 (Updated), 500 (Failed)

**IllustrationService:**
- Fields: 48 fields (owner info, insured info, coverage, riders, premiums, timestamps)
- Method: `CreateAndSaveIllustration`
- Complex: 3 dates (LocalDate), 3 datetimes (LocalDateTime)

## 🗣️ Questions to Ask Interviewer
1. "Does your team use gRPC for microservices?"
2. "How do you handle service versioning?"
3. "What's your observability strategy?"
4. "How do you test distributed systems?"

## ✅ Pre-Interview Checklist
- [ ] Can explain gRPC in 30 seconds
- [ ] Can write basic .proto file
- [ ] Can implement unary RPC
- [ ] Can write unit test
- [ ] Know StreamObserver rules
- [ ] Can discuss trade-offs
- [ ] Prepared project talking points
- [ ] Ready to ask questions

---

**🎓 Study Path:**
1. Read GRPC_RESUME_SUMMARY.md (10 min)
2. Study GRPC_INTERVIEW_GUIDE.md (2-3 hours)
3. Practice GRPC_CODING_EXERCISES.md (8-12 hours)
4. Review GRPC_SYSTEM_DESIGN.md (3-4 hours)
5. Memorize this cheat card (15 min)

**📞 Day-Of:**
- Review this card 30 min before
- Breathe, relax, be confident
- Be ready to whiteboard
- Ask clarifying questions

**Good luck! 🚀 You've got this! 💪**

---
*Print at 100% scale • Bring to interview • Quick reference*

