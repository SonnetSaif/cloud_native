provide me spring boot custom annotation with Gson serializer for this ProspectRequestModel class


Got it—here’s a clean, production-ready way to add a **custom annotation** that plugs into **Gson** for serializing your `ProspectRequestModel` in a Spring Boot app.

Below you’ll get:

1.  A custom annotation `@UseGsonSerializer`
2.  A `TypeAdapterFactory` that honors that annotation
3.  Spring Boot configuration to wire Gson properly
4.  A sample `ProspectRequestModel` using the annotation
5.  A sample serializer `ProspectRequestModelSerializer`
6.  (Optional) a simpler alternative using Gson’s built-in `@JsonAdapter` if you don’t need a custom annotation name

> ⚠️ Spring Boot uses **Jackson** by default. If you want to use **Gson** for your controllers’ JSON, you must configure it (see **Spring Boot config** section).

***

## 1) Create a custom annotation

```java
// src/main/java/com/example/json/UseGsonSerializer.java
package com.example.json;

import com.google.gson.JsonSerializer;
import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE) // class-level
@Documented
public @interface UseGsonSerializer {
    Class<? extends JsonSerializer<?>> value();
}
```

***

## 2) TypeAdapterFactory that applies the annotation

This factory detects `@UseGsonSerializer` on a class and uses the provided `JsonSerializer` for **serialization**, while delegating **deserialization** to Gson’s default.

```java
// src/main/java/com/example/json/UseGsonSerializerAdapterFactory.java
package com.example.json;

import com.google.gson.*;
import com.google.gson.internal.Streams;
import com.google.gson.reflect.TypeToken;
import com.google.gson.stream.JsonReader;
import com.google.gson.stream.JsonWriter;

import java.io.IOException;
import java.lang.reflect.Type;

public final class UseGsonSerializerAdapterFactory implements TypeAdapterFactory {

    @Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public <T> TypeAdapter<T> create(Gson gson, TypeToken<T> targetType) {
        Class<? super T> rawType = targetType.getRawType();
        UseGsonSerializer ann = rawType.getAnnotation(UseGsonSerializer.class);
        if (ann == null) {
            return null; // not our concern; let other factories handle it
        }

        final JsonSerializer serializer;
        try {
            serializer = ann.value().getDeclaredConstructor().newInstance();
        } catch (Exception e) {
            throw new IllegalStateException(
                "Failed to instantiate serializer: " + ann.value().getName(), e
            );
        }

        final TypeAdapter<T> delegate = gson.getDelegateAdapter(this, targetType);
        final Type type = targetType.getType();

        final JsonSerializationContext serializationContext = new JsonSerializationContext() {
            @Override public JsonElement serialize(Object src) {
                return gson.toJsonTree(src);
            }
            @Override public JsonElement serialize(Object src, Type t) {
                return gson.toJsonTree(src, t);
            }
        };

        return new TypeAdapter<T>() {
            @Override
            public void write(JsonWriter out, T value) throws IOException {
                if (value == null) {
                    out.nullValue();
                    return;
                }
                JsonElement tree = serializer.serialize(value, type, serializationContext);
                Streams.write(tree, out); // write JsonElement to writer
            }

            @Override
            public T read(JsonReader in) throws IOException {
                // Use default deserialization
                return delegate.read(in);
            }
        }.nullSafe();
    }
}
```

***

## 3) Spring Boot configuration to use Gson + register the factory

> Choose **one** approach depending on how your project is set up.

### A. Prefer Gson via application properties (Spring Boot 2.7+/3.x)

```properties
# src/main/resources/application.properties
spring.mvc.converters.preferred-json-mapper=gson
```

Then add a customizer bean:

```java
// src/main/java/com/example/config/GsonConfig.java
package com.example.config;

import com.example.json.UseGsonSerializerAdapterFactory;
import org.springframework.boot.autoconfigure.gson.GsonBuilderCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class GsonConfig {

    @Bean
    public GsonBuilderCustomizer customGson() {
        return builder -> builder.registerTypeAdapterFactory(new UseGsonSerializerAdapterFactory());
    }
}
```

### B. (Alternative) Provide a `Gson` bean + converter explicitly

```xml
<!-- Maven dependency -->
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
</dependency>
```

```java
// src/main/java/com/example/config/GsonConfig.java
package com.example.config;

import com.example.json.UseGsonSerializerAdapterFactory;
import com.google.gson.Gson;
import com.google.gson.GsonBuilder;
import org.springframework.boot.autoconfigure.http.HttpMessageConverters;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.converter.json.GsonHttpMessageConverter;

@Configuration
public class GsonConfig {

    @Bean
    public Gson gson() {
        return new GsonBuilder()
                .registerTypeAdapterFactory(new UseGsonSerializerAdapterFactory())
                .serializeNulls() // optional
                .create();
    }

    @Bean
    public HttpMessageConverters messageConverters(Gson gson) {
        GsonHttpMessageConverter converter = new GsonHttpMessageConverter();
        converter.setGson(gson);
        return new HttpMessageConverters(converter);
    }
}
```

> If Jackson is on your classpath and still taking precedence, either keep the property `spring.mvc.converters.preferred-json-mapper=gson` or exclude Jackson from your web starter.

***

## 4) Annotate your `ProspectRequestModel`

Apply your custom annotation and point to the serializer class.

```java
// src/main/java/com/example/model/ProspectRequestModel.java
package com.example.model;

import com.example.json.UseGsonSerializer;
import com.example.serializer.ProspectRequestModelSerializer;

import java.time.OffsetDateTime;
import java.util.Map;

@UseGsonSerializer(ProspectRequestModelSerializer.class)
public class ProspectRequestModel {

    private String prospectId;
    private String fullName;
    private String email;
    private OffsetDateTime createdAt;
    private Map<String, Object> metadata;

    // Getters/Setters/Constructors

    public String getProspectId() { return prospectId; }
    public void setProspectId(String prospectId) { this.prospectId = prospectId; }

    public String getFullName() { return fullName; }
    public void setFullName(String fullName) { this.fullName = fullName; }

    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }

    public OffsetDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(OffsetDateTime createdAt) { this.createdAt = createdAt; }

    public Map<String, Object> getMetadata() { return metadata; }
    public void setMetadata(Map<String, Object> metadata) { this.metadata = metadata; }
}
```

***

## 5) Implement the custom Gson serializer

Here’s a sample serializer that:

*   Controls field names,
*   Formats dates (ISO-8601),
*   Masks the email (example business rule),
*   Delegates complex/nested fields using the provided `JsonSerializationContext`.

```java
// src/main/java/com/example/serializer/ProspectRequestModelSerializer.java
package com.example.serializer;

import com.example.model.ProspectRequestModel;
import com.google.gson.JsonElement;
import com.google.gson.JsonObject;
import com.google.gson.JsonSerializationContext;
import com.google.gson.JsonSerializer;

import java.lang.reflect.Type;
import java.time.format.DateTimeFormatter;

public class ProspectRequestModelSerializer implements JsonSerializer<ProspectRequestModel> {

    private static final DateTimeFormatter ISO = DateTimeFormatter.ISO_OFFSET_DATE_TIME;

    @Override
    public JsonElement serialize(ProspectRequestModel src, Type typeOfSrc, JsonSerializationContext ctx) {
        JsonObject root = new JsonObject();

        // You choose how the JSON looks:
        root.addProperty("id", src.getProspectId());
        root.addProperty("name", src.getFullName());

        // Example: mask email for logs or outbound integrations
        if (src.getEmail() != null) {
            root.addProperty("email", maskEmail(src.getEmail()));
        } else {
            root.add("email", null);
        }

        if (src.getCreatedAt() != null) {
            root.addProperty("createdAt", ISO.format(src.getCreatedAt()));
        } else {
            root.add("createdAt", null);
        }

        // Delegate nested/complex structures to Gson
        if (src.getMetadata() != null) {
            root.add("metadata", ctx.serialize(src.getMetadata()));
        } else {
            root.add("metadata", null);
        }

        return root;
    }

    private String maskEmail(String email) {
        int at = email.indexOf('@');
        if (at <= 1) return "***"; // fallback
        String local = email.substring(0, at);
        String domain = email.substring(at);
        String visible = local.substring(0, Math.min(2, local.length()));
        return visible + "****" + domain;
    }
}
```

***

## 6) (Optional) Controller example

When your controller returns `ProspectRequestModel`, it’ll be serialized with your custom serializer:

```java
// src/main/java/com/example/web/ProspectController.java
package com.example.web;

import com.example.model.ProspectRequestModel;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.time.OffsetDateTime;
import java.util.Map;

@RestController
public class ProspectController {

    @GetMapping("/prospect/sample")
    public ProspectRequestModel sample() {
        ProspectRequestModel m = new ProspectRequestModel();
        m.setProspectId("P-12345");
        m.setFullName("Jane Doe");
        m.setEmail("jane.doe@example.com");
        m.setCreatedAt(OffsetDateTime.now());
        m.setMetadata(Map.of("source", "web", "score", 87));
        return m;
    }
}
```

***

## 7) Alternative (simpler): Use Gson’s built-in `@JsonAdapter`

If you don’t specifically need a custom annotation name, Gson already supports this:

```java
import com.google.gson.annotations.JsonAdapter;

@JsonAdapter(ProspectRequestModelTypeAdapter.class)
public class ProspectRequestModel {
   // fields...
}
```

Where `ProspectRequestModelTypeAdapter` is a `TypeAdapter<ProspectRequestModel>` (or `JsonSerializer`/`JsonDeserializer` wrapped in a `TypeAdapterFactory`). This avoids writing your own annotation and factory.

***

## Notes & Tips

*   If your project still serializes via Jackson, ensure:
    *   `spring.mvc.converters.preferred-json-mapper=gson` is set **or**
    *   Jackson is excluded from the classpath, **or**
    *   You explicitly register `GsonHttpMessageConverter` and ensure it’s effective.
*   You can extend the custom annotation to cover **deserialization** too by adding another class attribute for a `JsonDeserializer` and composing both into a single `TypeAdapter`.

***

If you paste your actual `ProspectRequestModel` fields (or any special rules you need—like date formats, null handling, field renames), I’ll tailor the serializer to your exact shape and requirements.
