@JsonInclude(JsonInclude.Include.NON_NULL)
public class Response<T> {

    private int status = 1000;
    private String message;

    private T data;

    private LocalDateTime requestDateTime;
    private LocalDateTime responseDateTime;

    public Response() {
    }

    public int getStatus() {
        return status;
    }

    public void setStatus(int status) {
        this.status = status;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public T getData() {
        return data;
    }

    public void setData(T data) {
        this.data = data;
    }

    public LocalDateTime getRequestDateTime() { return requestDateTime; }

    public void setRequestDateTime(LocalDateTime requestDateTime) {
        this.requestDateTime = requestDateTime;
    }

    public LocalDateTime getResponseDateTime() { return responseDateTime; }

    public void setResponseDateTime(LocalDateTime responseDateTime) {
        this.responseDateTime = responseDateTime;
    }

    public static class Builder<T> {

        private final Response<T> response;

        public Builder() {
            this.response = new Response<>();
        }

        public Builder<T> setStatus(int status) {
            this.response.setStatus(status);
            return this;
        }

        public Builder<T> setMessage(String message) {
            this.response.setMessage(message);
            return this;
        }

        public Builder<T> setData(T data) {
            this.response.setData(data);
            return this;
        }

        public Builder<T> setRequestDateTime(LocalDateTime requestDateTime) {
            this.response.setRequestDateTime(requestDateTime);
            return this;
        }

        public Builder<T> setResponseDateTime(LocalDateTime responseDateTime) {
            this.response.setResponseDateTime(responseDateTime);
            return this;
        }

        public Response<T> build() {
            return response;
        }
    }
}

explain me this

This is a **generic response wrapper class** in Java, commonly used in **REST APIs (e.g., Spring Boot)** to standardize how responses are sent to clients.

I’ll explain it **piece by piece**, then show **how it’s used**.

***

## 1. Purpose of the class

`Response<T>` is designed to return **structured API responses** like:

```json
{
  "status": 1000,
  "message": "Success",
  "data": {...},
  "requestDateTime": "2026-04-29T14:30:00",
  "responseDateTime": "2026-04-29T14:30:01"
}
```

It ensures **consistency** across all API endpoints.

***

## 2. Annotation: `@JsonInclude(JsonInclude.Include.NON_NULL)`

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
```

✅ This is a **Jackson annotation**

**What it does:**

*   Fields with `null` values **will not appear** in the JSON output.

Example:

```java
message = null
```

✅ `message` will be **excluded** from the JSON response.

This keeps responses **clean and smaller**.

***

## 3. Generic Type `<T>`

```java
public class Response<T>
```

`T` means **any type**.

So, `data` can be:

*   `User`
*   `List<User>`
*   `Map<String, Object>`
*   `String`

Examples:

```java
Response<User>
Response<List<User>>
Response<String>
```

***

## 4. Fields explained

```java
private int status = 1000;
```

*   Default status code
*   Often used as **business-level status**, not HTTP status
*   Example: `1000 = success`, `2001 = validation error`

***

```java
private String message;
```

*   Human-readable message
*   Example: `"Operation successful"`

***

```java
private T data;
```

*   The **actual response payload**
*   Generic so it works for any data type

***

```java
private LocalDateTime requestDateTime;
private LocalDateTime responseDateTime;
```

*   Tracks **when request started and finished**
*   Useful for:
    *   Logging
    *   Debugging
    *   Performance monitoring

***

## 5. Getters and Setters

All standard Java **POJO** methods:

```java
getStatus()
setStatus(int status)
```

✅ Required for:

*   Jackson serialization
*   Frameworks like Spring

***

## 6. Default Constructor

```java
public Response() {
}
```

*   Needed by Jackson and frameworks
*   Allows object creation via reflection

***

## 7. Builder Pattern (Inner `Builder<T>` class)

### Why use Builder?

Instead of:

```java
Response<User> r = new Response<>();
r.setStatus(1000);
r.setMessage("Success");
r.setData(user);
```

You can do:

```java
Response<User> r =
    new Response.Builder<User>()
        .setStatus(1000)
        .setMessage("Success")
        .setData(user)
        .setResponseDateTime(LocalDateTime.now())
        .build();
```

✅ Cleaner  
✅ Fluent  
✅ Less error-prone

***

### Builder internals

```java
private final Response<T> response;
```

*   Builder internally creates one `Response` instance
*   Each setter updates that object

***

Each builder method:

```java
public Builder<T> setStatus(int status) {
    this.response.setStatus(status);
    return this;
}
```

✅ Returns the builder itself → **method chaining**

***

Final build:

```java
public Response<T> build() {
    return response;
}
```

✅ Returns the fully-built response object

***

## 8. Typical usage in Spring Boot Controller

```java
@GetMapping("/user/{id}")
public Response<User> getUser(@PathVariable Long id) {
    User user = userService.findById(id);

    return new Response.Builder<User>()
            .setStatus(1000)
            .setMessage("User fetched successfully")
            .setData(user)
            .setRequestDateTime(LocalDateTime.now())
            .setResponseDateTime(LocalDateTime.now())
            .build();
}
```

***

## 9. Summary (in simple words)

✅ This class:

*   Standardizes API responses
*   Supports **any data type**
*   Avoids sending null fields
*   Uses **Builder Pattern** for clean object creation
*   Adds **timestamps** for tracking

✅ Common in:

*   Spring Boot
*   Microservices
*   Enterprise APIs

***

If you want, I can also:

*   Improve this design (immutability, Lombok, records)
*   Show how to integrate it with `ResponseEntity`
*   Add error handling examples

Just tell me 👍
