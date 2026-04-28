# gRPC Coding Exercises for Interview Preparation

## Table of Contents
1. [Beginner Exercises](#beginner-exercises)
2. [Intermediate Exercises](#intermediate-exercises)
3. [Advanced Exercises](#advanced-exercises)
4. [Real Interview Questions](#real-interview-questions)
5. [Live Coding Scenarios](#live-coding-scenarios)

---

## Beginner Exercises

### Exercise 1: Define a Simple gRPC Service

**Problem:** Define a gRPC service for a Calculator with add, subtract, multiply operations.

**Solution:**
```protobuf
syntax = "proto3";

package calculator;

service Calculator {
  rpc Add(BinaryOperation) returns (Result);
  rpc Subtract(BinaryOperation) returns (Result);
  rpc Multiply(BinaryOperation) returns (Result);
}

message BinaryOperation {
  double operand1 = 1;
  double operand2 = 2;
}

message Result {
  double value = 1;
  string error = 2;  // Optional error message
}
```

**Key Concepts:**
- Service definition with multiple RPC methods
- Message reuse (BinaryOperation for all operations)
- Error handling via optional field

---

### Exercise 2: Implement the Calculator Service

**Problem:** Implement the Calculator service in Java with Spring Boot.

**Solution:**
```java
@GrpcService
public class CalculatorGrpcService extends CalculatorGrpc.CalculatorImplBase {
    
    @Override
    public void add(BinaryOperation request, 
                   StreamObserver<Result> responseObserver) {
        double result = request.getOperand1() + request.getOperand2();
        
        Result response = Result.newBuilder()
            .setValue(result)
            .build();
        
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
    
    @Override
    public void subtract(BinaryOperation request,
                        StreamObserver<Result> responseObserver) {
        double result = request.getOperand1() - request.getOperand2();
        
        Result response = Result.newBuilder()
            .setValue(result)
            .build();
        
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
    
    @Override
    public void multiply(BinaryOperation request,
                        StreamObserver<Result> responseObserver) {
        // Handle division by zero scenario
        try {
            double result = request.getOperand1() * request.getOperand2();
            
            Result response = Result.newBuilder()
                .setValue(result)
                .build();
            
            responseObserver.onNext(response);
            responseObserver.onCompleted();
        } catch (Exception e) {
            Result errorResponse = Result.newBuilder()
                .setValue(0)
                .setError("Calculation error: " + e.getMessage())
                .build();
            
            responseObserver.onNext(errorResponse);
            responseObserver.onCompleted();
        }
    }
}
```

**Key Concepts:**
- Extending generated base class
- Using StreamObserver for responses
- Error handling without throwing exceptions

---

### Exercise 3: Write Unit Tests

**Problem:** Write unit tests for the Calculator service.

**Solution:**
```java
@ExtendWith(MockitoExtension.class)
class CalculatorGrpcServiceTest {
    
    @Mock
    private StreamObserver<Result> responseObserver;
    
    @InjectMocks
    private CalculatorGrpcService calculatorGrpcService;
    
    @Test
    void testAdd() {
        // Arrange
        BinaryOperation request = BinaryOperation.newBuilder()
            .setOperand1(5.0)
            .setOperand2(3.0)
            .build();
        
        // Act
        calculatorGrpcService.add(request, responseObserver);
        
        // Assert
        ArgumentCaptor<Result> captor = ArgumentCaptor.forClass(Result.class);
        verify(responseObserver).onNext(captor.capture());
        verify(responseObserver).onCompleted();
        
        Result result = captor.getValue();
        assertEquals(8.0, result.getValue(), 0.001);
        assertTrue(result.getError().isEmpty());
    }
    
    @Test
    void testSubtract() {
        BinaryOperation request = BinaryOperation.newBuilder()
            .setOperand1(10.0)
            .setOperand2(4.0)
            .build();
        
        calculatorGrpcService.subtract(request, responseObserver);
        
        ArgumentCaptor<Result> captor = ArgumentCaptor.forClass(Result.class);
        verify(responseObserver).onNext(captor.capture());
        verify(responseObserver).onCompleted();
        
        assertEquals(6.0, captor.getValue().getValue(), 0.001);
    }
}
```

**Key Concepts:**
- Mocking StreamObserver
- Using ArgumentCaptor to verify response
- Testing calculations with floating-point tolerance

---

## Intermediate Exercises

### Exercise 4: Add Server-Side Streaming

**Problem:** Implement a Fibonacci sequence generator that streams results.

**Proto Definition:**
```protobuf
service MathService {
  // Client sends count, server streams Fibonacci numbers
  rpc GenerateFibonacci(FibonacciRequest) returns (stream FibonacciNumber);
}

message FibonacciRequest {
  int32 count = 1;  // How many numbers to generate
}

message FibonacciNumber {
  int32 index = 1;
  int64 value = 2;
}
```

**Implementation:**
```java
@GrpcService
public class MathGrpcService extends MathServiceGrpc.MathServiceImplBase {
    
    @Override
    public void generateFibonacci(FibonacciRequest request,
                                  StreamObserver<FibonacciNumber> responseObserver) {
        int count = request.getCount();
        
        if (count <= 0) {
            responseObserver.onError(
                Status.INVALID_ARGUMENT
                    .withDescription("Count must be positive")
                    .asRuntimeException()
            );
            return;
        }
        
        long prev = 0, current = 1;
        
        // Send first number
        responseObserver.onNext(
            FibonacciNumber.newBuilder()
                .setIndex(0)
                .setValue(prev)
                .build()
        );
        
        // Stream remaining numbers
        for (int i = 1; i < count; i++) {
            responseObserver.onNext(
                FibonacciNumber.newBuilder()
                    .setIndex(i)
                    .setValue(current)
                    .build()
            );
            
            long next = prev + current;
            prev = current;
            current = next;
        }
        
        responseObserver.onCompleted();
    }
}
```

**Client Code:**
```java
public class FibonacciClient {
    public void printFibonacci(int count) {
        FibonacciRequest request = FibonacciRequest.newBuilder()
            .setCount(count)
            .build();
        
        Iterator<FibonacciNumber> numbers = 
            blockingStub.generateFibonacci(request);
        
        while (numbers.hasNext()) {
            FibonacciNumber num = numbers.next();
            System.out.printf("F(%d) = %d%n", num.getIndex(), num.getValue());
        }
    }
}
```

**Key Concepts:**
- Server-side streaming (one request, multiple responses)
- Calling `onNext()` multiple times
- Input validation with `onError()`
- Iterator pattern on client side

---

### Exercise 5: Add Client-Side Streaming

**Problem:** Implement a service that receives a stream of numbers and returns their average.

**Proto Definition:**
```protobuf
service StatsService {
  // Client streams numbers, server returns average
  rpc CalculateAverage(stream Number) returns (Average);
}

message Number {
  double value = 1;
}

message Average {
  double value = 1;
  int32 count = 2;
}
```

**Implementation:**
```java
@GrpcService
public class StatsGrpcService extends StatsServiceGrpc.StatsServiceImplBase {
    
    @Override
    public StreamObserver<Number> calculateAverage(
        StreamObserver<Average> responseObserver) {
        
        return new StreamObserver<Number>() {
            private double sum = 0;
            private int count = 0;
            
            @Override
            public void onNext(Number number) {
                // Accumulate each number as it arrives
                sum += number.getValue();
                count++;
            }
            
            @Override
            public void onError(Throwable t) {
                responseObserver.onError(t);
            }
            
            @Override
            public void onCompleted() {
                // Calculate and send average when client is done
                double average = count > 0 ? sum / count : 0;
                
                Average result = Average.newBuilder()
                    .setValue(average)
                    .setCount(count)
                    .build();
                
                responseObserver.onNext(result);
                responseObserver.onCompleted();
            }
        };
    }
}
```

**Client Code:**
```java
public class StatsClient {
    public void calculateAverage(List<Double> numbers) {
        StreamObserver<Average> responseObserver = new StreamObserver<>() {
            @Override
            public void onNext(Average average) {
                System.out.printf("Average of %d numbers: %.2f%n", 
                    average.getCount(), average.getValue());
            }
            
            @Override
            public void onCompleted() {
                System.out.println("Calculation complete");
            }
            
            @Override
            public void onError(Throwable t) {
                System.err.println("Error: " + t.getMessage());
            }
        };
        
        StreamObserver<Number> requestObserver = 
            asyncStub.calculateAverage(responseObserver);
        
        // Stream numbers to server
        for (Double num : numbers) {
            Number number = Number.newBuilder()
                .setValue(num)
                .build();
            requestObserver.onNext(number);
        }
        
        // Signal we're done sending
        requestObserver.onCompleted();
    }
}
```

**Key Concepts:**
- Client-side streaming (multiple requests, one response)
- Returning a StreamObserver for the server to receive data
- Accumulating state across multiple `onNext()` calls
- Async client pattern

---

### Exercise 6: Implement Bidirectional Streaming

**Problem:** Implement a chat service where multiple clients can send and receive messages.

**Proto Definition:**
```protobuf
service ChatService {
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message ChatMessage {
  string username = 1;
  string message = 2;
  int64 timestamp = 3;
}
```

**Implementation:**
```java
@GrpcService
public class ChatGrpcService extends ChatServiceGrpc.ChatServiceImplBase {
    
    private final Set<StreamObserver<ChatMessage>> observers = 
        ConcurrentHashMap.newKeySet();
    
    @Override
    public StreamObserver<ChatMessage> chat(
        StreamObserver<ChatMessage> responseObserver) {
        
        // Add this client to the broadcast list
        observers.add(responseObserver);
        
        return new StreamObserver<ChatMessage>() {
            @Override
            public void onNext(ChatMessage message) {
                // Broadcast to all connected clients
                ChatMessage timestamped = ChatMessage.newBuilder()
                    .setUsername(message.getUsername())
                    .setMessage(message.getMessage())
                    .setTimestamp(System.currentTimeMillis())
                    .build();
                
                for (StreamObserver<ChatMessage> observer : observers) {
                    try {
                        observer.onNext(timestamped);
                    } catch (Exception e) {
                        // Client disconnected, remove from list
                        observers.remove(observer);
                    }
                }
            }
            
            @Override
            public void onError(Throwable t) {
                observers.remove(responseObserver);
            }
            
            @Override
            public void onCompleted() {
                observers.remove(responseObserver);
                responseObserver.onCompleted();
            }
        };
    }
}
```

**Client Code:**
```java
public class ChatClient {
    public void joinChat(String username) {
        StreamObserver<ChatMessage> responseObserver = new StreamObserver<>() {
            @Override
            public void onNext(ChatMessage message) {
                System.out.printf("[%s] %s: %s%n",
                    new Date(message.getTimestamp()),
                    message.getUsername(),
                    message.getMessage());
            }
            
            @Override
            public void onCompleted() {
                System.out.println("Chat ended");
            }
            
            @Override
            public void onError(Throwable t) {
                System.err.println("Error: " + t.getMessage());
            }
        };
        
        StreamObserver<ChatMessage> requestObserver = 
            asyncStub.chat(responseObserver);
        
        // Read messages from console and send
        Scanner scanner = new Scanner(System.in);
        while (scanner.hasNextLine()) {
            String line = scanner.nextLine();
            if (line.equals("/quit")) break;
            
            ChatMessage message = ChatMessage.newBuilder()
                .setUsername(username)
                .setMessage(line)
                .build();
            
            requestObserver.onNext(message);
        }
        
        requestObserver.onCompleted();
    }
}
```

**Key Concepts:**
- Bidirectional streaming (full duplex)
- Managing multiple concurrent streams
- Thread-safe collections (ConcurrentHashMap.newKeySet())
- Broadcast pattern
- Error handling for disconnected clients

---

## Advanced Exercises

### Exercise 7: Implement Interceptors for Authentication

**Problem:** Create a server interceptor that validates JWT tokens.

**Solution:**
```java
@Component
public class JwtAuthServerInterceptor implements ServerInterceptor {
    
    private static final Metadata.Key<String> AUTHORIZATION = 
        Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);
    
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
        ServerCall<ReqT, RespT> call,
        Metadata headers,
        ServerCallHandler<ReqT, RespT> next) {
        
        String authHeader = headers.get(AUTHORIZATION);
        
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            call.close(Status.UNAUTHENTICATED
                .withDescription("Missing or invalid authorization header"), 
                new Metadata());
            return new ServerCall.Listener<>() {};
        }
        
        String token = authHeader.substring(7);
        
        try {
            // Validate JWT
            DecodedJWT jwt = JWT.require(Algorithm.HMAC256("secret"))
                .build()
                .verify(token);
            
            // Extract user info and add to context
            String userId = jwt.getClaim("userId").asString();
            Context context = Context.current()
                .withValue(USER_ID_KEY, userId);
            
            // Continue with authenticated context
            return Contexts.interceptCall(context, call, headers, next);
            
        } catch (JWTVerificationException e) {
            call.close(Status.UNAUTHENTICATED
                .withDescription("Invalid JWT token"), 
                new Metadata());
            return new ServerCall.Listener<>() {};
        }
    }
    
    // Context key for user ID
    public static final Context.Key<String> USER_ID_KEY = 
        Context.key("userId");
}

// Register globally
@Configuration
public class GrpcConfig {
    @Bean
    public GlobalServerInterceptorConfigurer globalInterceptor(
        JwtAuthServerInterceptor jwtInterceptor) {
        
        return registry -> registry.addServerInterceptors(jwtInterceptor);
    }
}

// Or register per-service
@GrpcService(interceptors = JwtAuthServerInterceptor.class)
public class ProspectGrpcService extends ... {
    
    @Override
    public void createAndSaveProspect(...) {
        // Access user ID from context
        String userId = JwtAuthServerInterceptor.USER_ID_KEY.get();
        System.out.println("Request from user: " + userId);
        
        // ... rest of implementation
    }
}
```

**Client-Side:**
```java
public class AuthenticatedClient {
    public void makeAuthenticatedCall(String jwtToken) {
        Metadata headers = new Metadata();
        headers.put(
            Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER),
            "Bearer " + jwtToken
        );
        
        ProspectServiceBlockingStub stub = MetadataUtils.attachHeaders(
            ProspectServiceGrpc.newBlockingStub(channel),
            headers
        );
        
        CreateAndSaveProspectResponse response = 
            stub.createAndSaveProspect(request);
    }
}
```

**Key Concepts:**
- Server interceptors for cross-cutting concerns
- Metadata for passing headers
- Context propagation
- JWT validation
- Global vs per-service interceptors

---

### Exercise 8: Implement Retry Logic

**Problem:** Implement client-side retry logic with exponential backoff.

**Solution:**
```java
public class ResilientGrpcClient {
    private final ProspectServiceBlockingStub stub;
    private final int maxRetries = 3;
    private final long initialBackoffMs = 100;
    
    public CreateAndSaveProspectResponse createProspectWithRetry(
        CreateAndSaveProspectRequest request) {
        
        int attempt = 0;
        long backoff = initialBackoffMs;
        
        while (attempt < maxRetries) {
            try {
                return stub
                    .withDeadlineAfter(5, TimeUnit.SECONDS)
                    .createAndSaveProspect(request);
                    
            } catch (StatusRuntimeException e) {
                attempt++;
                
                if (!isRetryable(e.getStatus()) || attempt >= maxRetries) {
                    throw e;  // Don't retry, rethrow
                }
                
                System.out.printf("Attempt %d failed: %s. Retrying in %dms...%n",
                    attempt, e.getStatus(), backoff);
                
                try {
                    Thread.sleep(backoff);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw e;
                }
                
                // Exponential backoff: 100ms, 200ms, 400ms, ...
                backoff *= 2;
            }
        }
        
        throw new RuntimeException("Max retries exceeded");
    }
    
    private boolean isRetryable(Status status) {
        // Retry on transient errors
        return status.getCode() == Status.Code.UNAVAILABLE
            || status.getCode() == Status.Code.DEADLINE_EXCEEDED
            || status.getCode() == Status.Code.RESOURCE_EXHAUSTED;
    }
}
```

**Advanced: Circuit Breaker Pattern:**
```java
public class CircuitBreakerGrpcClient {
    private enum State { CLOSED, OPEN, HALF_OPEN }
    
    private State state = State.CLOSED;
    private int failureCount = 0;
    private final int failureThreshold = 5;
    private long openStateStartTime = 0;
    private final long openStateDurationMs = 30000;  // 30 seconds
    
    public CreateAndSaveProspectResponse createProspect(
        CreateAndSaveProspectRequest request) {
        
        if (state == State.OPEN) {
            if (System.currentTimeMillis() - openStateStartTime > openStateDurationMs) {
                state = State.HALF_OPEN;
                failureCount = 0;
            } else {
                throw new RuntimeException("Circuit breaker is OPEN");
            }
        }
        
        try {
            CreateAndSaveProspectResponse response = 
                stub.createAndSaveProspect(request);
            
            // Success
            if (state == State.HALF_OPEN) {
                state = State.CLOSED;
            }
            failureCount = 0;
            return response;
            
        } catch (StatusRuntimeException e) {
            failureCount++;
            
            if (failureCount >= failureThreshold) {
                state = State.OPEN;
                openStateStartTime = System.currentTimeMillis();
                System.out.println("Circuit breaker opened!");
            }
            
            throw e;
        }
    }
}
```

**Key Concepts:**
- Retry logic with exponential backoff
- Identifying retryable vs non-retryable errors
- Circuit breaker pattern
- State management
- Deadline configuration

---

## Real Interview Questions

### Question 1: How would you handle large file uploads in gRPC?

**Answer:**
```protobuf
service FileUploadService {
  // Client streams file chunks
  rpc UploadFile(stream FileChunk) returns (UploadStatus);
}

message FileChunk {
  string filename = 1;
  bytes content = 2;
  int32 chunk_number = 3;
}

message UploadStatus {
  bool success = 1;
  string message = 2;
  int64 total_bytes = 3;
}
```

**Implementation:**
```java
@GrpcService
public class FileUploadGrpcService extends FileUploadServiceGrpc.FileUploadServiceImplBase {
    
    @Override
    public StreamObserver<FileChunk> uploadFile(
        StreamObserver<UploadStatus> responseObserver) {
        
        return new StreamObserver<FileChunk>() {
            private FileOutputStream outputStream;
            private String filename;
            private long totalBytes = 0;
            
            @Override
            public void onNext(FileChunk chunk) {
                try {
                    if (outputStream == null) {
                        filename = chunk.getFilename();
                        outputStream = new FileOutputStream("/uploads/" + filename);
                    }
                    
                    byte[] data = chunk.getContent().toByteArray();
                    outputStream.write(data);
                    totalBytes += data.length;
                    
                } catch (IOException e) {
                    onError(e);
                }
            }
            
            @Override
            public void onError(Throwable t) {
                cleanup();
                responseObserver.onError(
                    Status.INTERNAL
                        .withDescription("Upload failed: " + t.getMessage())
                        .asRuntimeException()
                );
            }
            
            @Override
            public void onCompleted() {
                cleanup();
                
                UploadStatus status = UploadStatus.newBuilder()
                    .setSuccess(true)
                    .setMessage("File uploaded successfully")
                    .setTotalBytes(totalBytes)
                    .build();
                
                responseObserver.onNext(status);
                responseObserver.onCompleted();
            }
            
            private void cleanup() {
                if (outputStream != null) {
                    try {
                        outputStream.close();
                    } catch (IOException e) {
                        // Log error
                    }
                }
            }
        };
    }
}
```

**Client:**
```java
public class FileUploadClient {
    public void uploadFile(String filePath) throws IOException {
        File file = new File(filePath);
        
        StreamObserver<UploadStatus> responseObserver = new StreamObserver<>() {
            @Override
            public void onNext(UploadStatus status) {
                System.out.printf("Upload complete: %d bytes%n", 
                    status.getTotalBytes());
            }
            
            @Override
            public void onCompleted() {
                System.out.println("Done");
            }
            
            @Override
            public void onError(Throwable t) {
                System.err.println("Upload failed: " + t.getMessage());
            }
        };
        
        StreamObserver<FileChunk> requestObserver = 
            asyncStub.uploadFile(responseObserver);
        
        try (FileInputStream fis = new FileInputStream(file)) {
            byte[] buffer = new byte[4096];  // 4KB chunks
            int bytesRead;
            int chunkNumber = 0;
            
            while ((bytesRead = fis.read(buffer)) != -1) {
                FileChunk chunk = FileChunk.newBuilder()
                    .setFilename(file.getName())
                    .setContent(ByteString.copyFrom(buffer, 0, bytesRead))
                    .setChunkNumber(chunkNumber++)
                    .build();
                
                requestObserver.onNext(chunk);
            }
            
            requestObserver.onCompleted();
        }
    }
}
```

**Key Concepts:**
- Client-side streaming for large data
- Chunking strategy (4KB chunks)
- Resource cleanup (file handles)
- Error handling during streaming

---

### Question 2: How do you implement pagination in gRPC?

**Answer:**
```protobuf
service UserService {
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
}

message ListUsersRequest {
  int32 page_size = 1;    // How many per page
  string page_token = 2;  // Token for next page (opaque to client)
}

message ListUsersResponse {
  repeated User users = 1;
  string next_page_token = 2;  // Empty if no more pages
  int32 total_count = 3;
}

message User {
  string id = 1;
  string name = 2;
}
```

**Implementation:**
```java
@GrpcService
public class UserGrpcService extends UserServiceGrpc.UserServiceImplBase {
    
    @Override
    public void listUsers(ListUsersRequest request,
                         StreamObserver<ListUsersResponse> responseObserver) {
        int pageSize = request.getPageSize();
        if (pageSize <= 0) pageSize = 10;  // Default page size
        
        // Decode page token (could be offset, cursor, etc.)
        int offset = 0;
        if (!request.getPageToken().isEmpty()) {
            offset = Integer.parseInt(
                new String(Base64.getDecoder().decode(request.getPageToken()))
            );
        }
        
        // Fetch from database
        List<UserEntity> entities = userRepository.findAll(
            PageRequest.of(offset / pageSize, pageSize)
        ).getContent();
        
        // Convert to proto messages
        List<User> users = entities.stream()
            .map(entity -> User.newBuilder()
                .setId(entity.getId())
                .setName(entity.getName())
                .build())
            .collect(Collectors.toList());
        
        // Generate next page token
        String nextPageToken = "";
        if (entities.size() == pageSize) {  // More pages available
            int nextOffset = offset + pageSize;
            nextPageToken = Base64.getEncoder().encodeToString(
                String.valueOf(nextOffset).getBytes()
            );
        }
        
        // Build response
        ListUsersResponse response = ListUsersResponse.newBuilder()
            .addAllUsers(users)
            .setNextPageToken(nextPageToken)
            .setTotalCount((int) userRepository.count())
            .build();
        
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
}
```

**Client:**
```java
public class UserClient {
    public void listAllUsers() {
        String pageToken = "";
        int pageCount = 0;
        
        do {
            ListUsersRequest request = ListUsersRequest.newBuilder()
                .setPageSize(10)
                .setPageToken(pageToken)
                .build();
            
            ListUsersResponse response = stub.listUsers(request);
            
            System.out.printf("Page %d: %d users%n", 
                ++pageCount, response.getUsersCount());
            
            for (User user : response.getUsersList()) {
                System.out.println("  - " + user.getName());
            }
            
            pageToken = response.getNextPageToken();
            
        } while (!pageToken.isEmpty());
        
        System.out.println("Total pages: " + pageCount);
    }
}
```

**Key Concepts:**
- Page tokens (opaque, base64-encoded)
- Default page size
- Total count for UI
- Cursor-based vs offset-based pagination

---

## Live Coding Scenarios

### Scenario 1: Debug a StreamObserver Memory Leak

**Given Code:**
```java
@GrpcService
public class BuggyService extends SomeServiceGrpc.SomeServiceImplBase {
    @Override
    public void processRequest(Request request, 
                              StreamObserver<Response> responseObserver) {
        try {
            // Process request
            Response response = buildResponse(request);
            responseObserver.onNext(response);
            // BUG: Missing onCompleted() call
        } catch (Exception e) {
            Response errorResponse = buildErrorResponse(e);
            responseObserver.onNext(errorResponse);
            // BUG: Missing onCompleted() here too
        }
    }
}
```

**Fix:**
```java
@GrpcService
public class FixedService extends SomeServiceGrpc.SomeServiceImplBase {
    @Override
    public void processRequest(Request request,
                              StreamObserver<Response> responseObserver) {
        try {
            Response response = buildResponse(request);
            responseObserver.onNext(response);
            responseObserver.onCompleted();  // ✅ Added
        } catch (Exception e) {
            Response errorResponse = buildErrorResponse(e);
            responseObserver.onNext(errorResponse);
            responseObserver.onCompleted();  // ✅ Added
        }
    }
}
```

**Explanation:**
- Always call `onCompleted()` or `onError()` to release resources
- Without it, streams stay open, consuming memory
- Use finally block for guarantee

---

### Scenario 2: Add Timeout Handling

**Given Code:**
```java
// Client making slow calls
public Response makeCall(Request request) {
    return stub.processRequest(request);  // May hang forever
}
```

**Fix:**
```java
public Response makeCallWithTimeout(Request request) {
    try {
        return stub
            .withDeadlineAfter(5, TimeUnit.SECONDS)  // ✅ Add timeout
            .processRequest(request);
    } catch (StatusRuntimeException e) {
        if (e.getStatus().getCode() == Status.Code.DEADLINE_EXCEEDED) {
            System.err.println("Request timed out after 5 seconds");
            // Handle timeout (retry, fallback, etc.)
        }
        throw e;
    }
}
```

---

### Scenario 3: Convert REST Endpoint to gRPC

**Given REST API:**
```java
@RestController
public class UserController {
    @PostMapping("/api/users")
    public ResponseEntity<UserResponse> createUser(@RequestBody UserRequest request) {
        User user = userService.createUser(
            request.getName(),
            request.getEmail()
        );
        
        UserResponse response = new UserResponse(
            user.getId(),
            user.getName(),
            user.getEmail(),
            "Created successfully"
        );
        
        return ResponseEntity.status(201).body(response);
    }
}
```

**Convert to gRPC:**

**Step 1: Define Proto**
```protobuf
service UserService {
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message CreateUserResponse {
  string id = 1;
  string name = 2;
  string email = 3;
  string message = 4;
}
```

**Step 2: Implement Service**
```java
@GrpcService
public class UserGrpcService extends UserServiceGrpc.UserServiceImplBase {
    
    private final UserService userService;
    
    @Override
    public void createUser(CreateUserRequest request,
                          StreamObserver<CreateUserResponse> responseObserver) {
        User user = userService.createUser(
            request.getName(),
            request.getEmail()
        );
        
        CreateUserResponse response = CreateUserResponse.newBuilder()
            .setId(user.getId())
            .setName(user.getName())
            .setEmail(user.getEmail())
            .setMessage("Created successfully")
            .build();
        
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
}
```

---

## Practice Problems

### Problem 1: Build a Rate Limiter
Implement a gRPC interceptor that limits clients to 10 requests per minute.

### Problem 2: Add Metrics Collection
Add Application Insights tracking to measure:
- Request count
- Average latency
- Error rate

### Problem 3: Implement Health Checks
Add gRPC health check service following the standard protocol.

### Problem 4: Build a Load Balancer
Implement client-side load balancing across multiple server endpoints.

### Problem 5: Add Request Logging
Create an interceptor that logs all requests with:
- Timestamp
- Method name
- Client IP
- Request duration

---

**Document Purpose:** Hands-on coding practice for gRPC interviews  
**Difficulty Progression:** Beginner → Intermediate → Advanced → Real-world  
**Estimated Practice Time:** 8-12 hours to complete all exercises

