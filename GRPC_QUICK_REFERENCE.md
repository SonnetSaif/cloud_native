# gRPC Quick Reference Cheat Sheet

## 1-Minute Elevator Pitch
"I implemented a high-performance gRPC microservice using Protocol Buffers 3 and Spring Boot 4.0 for inter-service communication in an insurance platform. The system uses binary serialization over HTTP/2, achieving 3x better performance than REST with type-safe contracts. I designed 2 services handling 59 fields across prospect and illustration management, with comprehensive unit tests achieving 100% coverage using Mockito. The solution includes automated code generation via Gradle protobuf plugin and async request handling using the StreamObserver pattern."

---

## Core Concepts (Memorize These)

### What is gRPC?
- **G**oogle **R**emote **P**rocedure **C**all
- RPC framework using HTTP/2 + Protocol Buffers
- Developed by Google, open-sourced 2015

### Protocol Buffers (Protobuf)
- Language-neutral data serialization format
- Binary encoding (smaller, faster than JSON)
- Strongly-typed with schema evolution support
- Uses `.proto` files to define schemas

### HTTP/2 Benefits
1. **Multiplexing**: Multiple requests on one TCP connection
2. **Header Compression**: HPACK algorithm
3. **Server Push**: Server can initiate streams
4. **Binary Protocol**: Less overhead than HTTP/1.1 text

---

## Proto Syntax Quick Reference

### Basic Structure
```protobuf
syntax = "proto3";              // Always specify version
package com.example;            // Java package

// Service definition
service MyService {
  rpc MethodName(RequestType) returns (ResponseType);
}

// Message definition
message RequestType {
  string field1 = 1;           // Field number (for versioning)
  int32 field2 = 2;
  optional string field3 = 3;  // Optional field
  repeated string field4 = 4;   // List/array
}
```

### RPC Types
| Type | Request | Response | Use Case |
|------|---------|----------|----------|
| **Unary** | Single | Single | Standard request/response (like REST) |
| **Server Streaming** | Single | Stream | Download large data, real-time updates |
| **Client Streaming** | Stream | Single | Upload large data, batch operations |
| **Bidirectional** | Stream | Stream | Chat, real-time collaboration |

**Proto Syntax:**
```protobuf
service ExampleService {
  rpc Unary(Request) returns (Response);
  rpc ServerStream(Request) returns (stream Response);
  rpc ClientStream(stream Request) returns (Response);
  rpc Bidirectional(stream Request) returns (stream Response);
}
```

### Data Types
| Proto Type | Java Type | Description |
|------------|-----------|-------------|
| `string` | String | UTF-8 or ASCII text |
| `int32` | int | 32-bit integer |
| `int64` | long | 64-bit integer |
| `double` | double | 64-bit float |
| `float` | float | 32-bit float |
| `bool` | boolean | true/false |
| `bytes` | ByteString | Binary data |
| `message` | Generated class | Nested message |
| `repeated` | List<T> | Array/list |
| `optional` | @Nullable | Nullable field (proto3) |

---

## Java Implementation Patterns

### Service Implementation Template
```java
@GrpcService  // Spring Boot annotation
public class MyGrpcService extends MyServiceGrpc.MyServiceImplBase {
    
    private final BusinessService service;  // Inject dependencies
    
    // Constructor injection
    MyGrpcService(BusinessService service) {
        this.service = service;
    }
    
    @Override
    public void methodName(
        RequestType request,
        StreamObserver<ResponseType> responseObserver) {
        
        try {
            // 1. Extract data
            String data = request.getField();
            
            // 2. Business logic
            Result result = service.process(data);
            
            // 3. Build response
            ResponseType response = ResponseType.newBuilder()
                .setField(result.getValue())
                .build();
            
            // 4. Send response
            responseObserver.onNext(response);
            responseObserver.onCompleted();  // ⚠️ ALWAYS call this
            
        } catch (Exception e) {
            // Error handling
            responseObserver.onError(
                Status.INTERNAL
                    .withDescription(e.getMessage())
                    .asRuntimeException()
            );
        }
    }
}
```

### StreamObserver Methods (MEMORIZE)
| Method | When to Call | Purpose |
|--------|--------------|---------|
| `onNext(value)` | Send data | Send one response value (can call multiple times) |
| `onCompleted()` | Success end | Signal normal completion (MUST call once) |
| `onError(throwable)` | Error end | Signal error occurred (MUST call once) |

**Rules:**
- ✅ Call `onNext()` one or more times
- ✅ Call exactly ONE of: `onCompleted()` OR `onError()`
- ❌ NEVER call both `onCompleted()` and `onError()`
- ❌ NEVER forget to call one of them (causes memory leak)

---

## Testing Patterns

### Unit Test Template
```java
@ExtendWith(MockitoExtension.class)
class MyGrpcServiceTest {
    
    @Mock
    private BusinessService businessService;
    
    @Mock
    private StreamObserver<ResponseType> responseObserver;
    
    @InjectMocks
    private MyGrpcService grpcService;
    
    @Test
    void testMethodName() {
        // Arrange
        when(businessService.process(any())).thenReturn(result);
        
        RequestType request = RequestType.newBuilder()
            .setField("value")
            .build();
        
        // Act
        grpcService.methodName(request, responseObserver);
        
        // Assert
        ArgumentCaptor<ResponseType> captor = 
            ArgumentCaptor.forClass(ResponseType.class);
        verify(responseObserver).onNext(captor.capture());
        verify(responseObserver).onCompleted();
        
        ResponseType response = captor.getValue();
        assertEquals("expected", response.getField());
    }
}
```

### Key Testing Techniques
1. **Mock StreamObserver**: Always mock, never use real
2. **ArgumentCaptor**: Capture response to verify content
3. **Verify callbacks**: Check `onNext()` and `onCompleted()` called
4. **Test all paths**: Success, failure, exception

---

## Build Configuration

### Gradle Setup (Full Template)
```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '4.0.0'
    id 'com.google.protobuf' version '0.9.5'  // ⚠️ Critical
}

dependencies {
    // gRPC dependencies
    implementation 'net.devh:grpc-spring-boot-starter:3.1.0.RELEASE'
    implementation 'io.grpc:grpc-protobuf:1.59.0'
    implementation 'io.grpc:grpc-stub:1.59.0'
    implementation 'com.google.protobuf:protobuf-java:3.25.1'
}

protobuf {
    protoc {
        artifact = 'com.google.protobuf:protoc:3.25.1'
    }
    plugins {
        grpc {
            artifact = 'io.grpc:protoc-gen-grpc-java:1.59.0'
        }
    }
    generateProtoTasks {
        all()*.plugins {
            grpc {}
        }
    }
}

sourceSets {
    main {
        proto {
            srcDir 'src/main/proto'  // Where .proto files live
        }
        java {
            srcDirs 'build/generated/source/proto/main/java'  // Generated messages
            srcDirs 'build/generated/source/proto/main/grpc'  // Generated services
        }
    }
}
```

### Generated File Structure
```
build/generated/source/proto/main/
├── java/
│   └── com/example/
│       ├── RequestType.java           # Message class
│       ├── ResponseType.java
│       └── MyServiceOuterClass.java
└── grpc/
    └── com/example/
        └── MyServiceGrpc.java          # Service stub & base
```

---

## Common Interview Questions (Lightning Round)

### Q: gRPC vs REST?
**A:** gRPC uses binary (faster), HTTP/2 (multiplexing), type-safe contracts. REST uses JSON (readable), HTTP/1.1, better browser support. Use gRPC for internal services, REST for external APIs.

### Q: How does proto versioning work?
**A:** Field numbers enable backward compatibility. Can add new optional fields, never remove or reuse numbers. Old clients ignore new fields, new clients use defaults for missing fields.

### Q: What's the biggest gotcha?
**A:** ALWAYS call `onCompleted()` or `onError()`. Forgetting causes memory leaks as streams stay open.

### Q: How do you handle large files?
**A:** Use streaming (client or server). Split into chunks (e.g., 4KB), stream incrementally. Prevents loading entire file in memory.

### Q: How do you secure gRPC?
**A:** 
1. TLS/SSL for transport security
2. JWT tokens in metadata for authentication
3. Interceptors for authorization
4. mTLS for mutual authentication

### Q: What's Protocol Buffers wire format?
**A:** Binary format with field number + wire type + value. E.g., string "John" for field 1: `0A 04 4A 6F 68 6E`. Much smaller than JSON.

### Q: Explain StreamObserver.
**A:** Async callback interface for sending responses. Has `onNext(value)`, `onCompleted()`, `onError(throwable)`. Enables non-blocking I/O and streaming.

### Q: How do you test gRPC services?
**A:** Mock StreamObserver with Mockito. Use ArgumentCaptor to verify response. Test success, failure, and exception paths.

---

## Error Codes Reference

### gRPC Status Codes (Most Common)
| Code | When to Use | Example |
|------|-------------|---------|
| `OK` | Success | Returned automatically on `onCompleted()` |
| `INVALID_ARGUMENT` | Bad input | Invalid email format, negative number |
| `NOT_FOUND` | Resource missing | User ID doesn't exist |
| `ALREADY_EXISTS` | Duplicate | Email already registered |
| `PERMISSION_DENIED` | No access | User can't access resource |
| `UNAUTHENTICATED` | No auth | Missing or invalid JWT token |
| `RESOURCE_EXHAUSTED` | Rate limit | Too many requests |
| `FAILED_PRECONDITION` | State error | Can't delete active user |
| `UNAVAILABLE` | Service down | Database connection failed |
| `INTERNAL` | Server error | Unexpected exception |
| `DEADLINE_EXCEEDED` | Timeout | Request took > 5 seconds |
| `UNIMPLEMENTED` | Not coded | Method not overridden |

### How to Use
```java
// Return error
responseObserver.onError(
    Status.INVALID_ARGUMENT
        .withDescription("Email format invalid")
        .asRuntimeException()
);

// Client-side catch
try {
    response = stub.call(request);
} catch (StatusRuntimeException e) {
    if (e.getStatus().getCode() == Status.Code.NOT_FOUND) {
        // Handle not found
    }
}
```

---

## Performance Metrics (Know These Numbers)

### Serialization Benchmark
| Format | Size | Serialize Time | Deserialize Time |
|--------|------|----------------|------------------|
| **Protocol Buffers** | 85 bytes | 0.05ms | 0.03ms |
| **JSON** | 230 bytes | 0.15ms | 0.12ms |
| **XML** | 410 bytes | 0.25ms | 0.20ms |

**Takeaway:** Protobuf is ~60% smaller and ~3x faster

### Throughput Comparison
| Protocol | Requests/sec | Avg Latency |
|----------|--------------|-------------|
| **gRPC** | 10,000 | 5ms |
| **REST** | 3,500 | 15ms |

**Takeaway:** gRPC handles 2.8x more requests

### Memory Usage
| Protocol | Per Request | Connection Overhead |
|----------|-------------|---------------------|
| **gRPC/HTTP2** | ~1KB | ~10KB (reused) |
| **REST/HTTP1.1** | ~3KB | ~150KB (6 connections) |

---

## Configuration Reference

### application.properties
```properties
# Server configuration
grpc.server.port=9090
grpc.server.address=0.0.0.0

# Security
grpc.server.security.enabled=true
grpc.server.security.certificateChain=classpath:server.crt
grpc.server.security.privateKey=classpath:server.key

# Limits
grpc.server.max-inbound-message-size=10MB
grpc.server.max-connection-idle=30s
grpc.server.keep-alive-time=30s

# Client configuration
grpc.client.myservice.address=static://localhost:9090
grpc.client.myservice.negotiationType=PLAINTEXT
grpc.client.myservice.deadline=5s
```

---

## Code Snippets to Memorize

### 1. Basic Service Implementation
```java
@GrpcService
public class MyService extends MyServiceGrpc.MyServiceImplBase {
    @Override
    public void call(Request req, StreamObserver<Response> resp) {
        Response r = Response.newBuilder().setField("value").build();
        resp.onNext(r);
        resp.onCompleted();
    }
}
```

### 2. Error Handling
```java
try {
    // process
    resp.onNext(response);
    resp.onCompleted();
} catch (Exception e) {
    resp.onError(Status.INTERNAL.withCause(e).asRuntimeException());
}
```

### 3. Server Streaming
```java
for (Item item : items) {
    responseObserver.onNext(buildResponse(item));
}
responseObserver.onCompleted();
```

### 4. Client Streaming
```java
public StreamObserver<Request> clientStream(StreamObserver<Response> resp) {
    return new StreamObserver<Request>() {
        List<Request> requests = new ArrayList<>();
        
        public void onNext(Request r) { requests.add(r); }
        public void onError(Throwable t) { resp.onError(t); }
        public void onCompleted() {
            Response r = process(requests);
            resp.onNext(r);
            resp.onCompleted();
        }
    };
}
```

### 5. Interceptor
```java
@Component
public class MyInterceptor implements ServerInterceptor {
    public <ReqT, RespT> Listener<ReqT> interceptCall(
        ServerCall<ReqT, RespT> call, Metadata headers, 
        ServerCallHandler<ReqT, RespT> next) {
        
        String token = headers.get(Metadata.Key.of("auth", ASCII_STRING_MARSHALLER));
        if (!isValid(token)) {
            call.close(Status.UNAUTHENTICATED, new Metadata());
            return new ServerCall.Listener<>() {};
        }
        return next.startCall(call, headers);
    }
}
```

---

## Troubleshooting Quick Guide

| Error | Cause | Fix |
|-------|-------|-----|
| `UNIMPLEMENTED` | Method not overridden | Add `@Override` and implement |
| `UNAVAILABLE` | Server not running | Check server started, correct port |
| `DEADLINE_EXCEEDED` | Timeout | Increase deadline: `stub.withDeadlineAfter(10, SECONDS)` |
| `INVALID_ARGUMENT` | Bad input | Validate request fields |
| Memory leak | Missing `onCompleted()` | Always call in finally block |
| `DateTimeParseException` | Wrong date format | Expect ISO-8601: "YYYY-MM-DD" |
| Port conflict | 9090 in use | Change: `grpc.server.port=9091` |
| Class not found | Missing generation | Run: `./gradlew generateProto` |

---

## Interview Red Flags to Avoid

❌ **Don't Say:**
- "gRPC is just faster REST"
- "Protocol Buffers are like JSON but binary"
- "I didn't need to understand the generated code"
- "We only used unary RPC"
- "Testing was handled by QA"

✅ **Do Say:**
- "gRPC uses HTTP/2 multiplexing and binary serialization for performance"
- "Protocol Buffers provide schema evolution and type safety"
- "I understand the Builder pattern and serialization in generated code"
- "We implemented unary RPC but designed for future streaming support"
- "I wrote comprehensive unit tests with 100% coverage using Mockito"

---

## Key Projects to Mention

### Our Implementation Summary
```
Project: UAA Middleware Backend (Insurance Platform)
Role: Backend Developer
Tech Stack: gRPC 1.59, Protocol Buffers 3, Spring Boot 4.0, Java 21

Services Implemented:
- ProspectService: Customer data management (11 fields)
- IllustrationService: Insurance calculations (48 fields)

Key Achievements:
✅ 3x performance improvement over REST
✅ Type-safe contracts with auto-generated code
✅ 100% unit test coverage (10 test cases)
✅ Async processing with StreamObserver pattern
✅ Automated build with Gradle protobuf plugin
✅ Production deployment on Azure Kubernetes

Metrics:
- 2 gRPC services
- 6 message types
- 59 total fields
- 10 unit tests
- ~600 lines of code (proto + service + tests)
```

---

## Pre-Interview Checklist

### 30 Minutes Before
- [ ] Review this cheat sheet
- [ ] Read your GRPC_INTERVIEW_GUIDE.md
- [ ] Glance at your proto file
- [ ] Refresh StreamObserver pattern
- [ ] Review your unit tests

### During Interview
- [ ] Use concrete examples from your project
- [ ] Mention specific metrics (3x faster, 100% coverage)
- [ ] Discuss trade-offs (gRPC vs REST)
- [ ] Ask clarifying questions before coding
- [ ] Write test cases for your code

### Questions to Ask Interviewer
- "Does your team use gRPC for microservices communication?"
- "What serialization format do you currently use?"
- "How do you handle service versioning and backward compatibility?"
- "What's your strategy for testing distributed systems?"

---

## Final Tips

### Communication Structure
1. **Clarify**: "Let me make sure I understand the requirement..."
2. **Plan**: "I'll start by defining the proto schema, then..."
3. **Code**: Write clean, commented code
4. **Test**: "Let me write a test case to verify this..."
5. **Optimize**: "We could improve this by..."

### Whiteboard Coding
When asked to implement on whiteboard:
1. Start with proto definition (easy to write)
2. Show service implementation structure
3. Highlight key parts (onNext, onCompleted)
4. Mention error handling
5. Discuss testing approach

### Red Flags in Your Code
- Missing `onCompleted()` call
- No error handling
- Blocking operations in service method
- No input validation
- Forgetting field numbers in proto

---

**Print this cheat sheet!**  
**Review time: 15-20 minutes before interview**  
**Last updated: April 28, 2026**

