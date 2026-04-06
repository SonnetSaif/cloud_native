@PostMapping("/submit")
public Response<String> saveApplication(@RequestBody @Valid ApplicationRequestDto applicationRequestDto) {

        applicationService.submitApplication(applicationRequestDto);

        return new Response.Builder<String>()
                .setStatus(1000)
                .setMessage("Success")
                .build();
    }how to if a parameter named 'Channel' is 'Bank' then the validation will work otherwise won't.how to handle?

    You’ve got two clean ways to make **validation run only when `channel == "Bank"`**:

***

## ✅ Option A — Dynamic groups with **Hibernate Validator** (`@GroupSequenceProvider`)

This is the most declarative and keeps your controller simple.

### 1) Define a validation group

```java
public interface BankGroup {}
```

### 2) Annotate fields that should validate **only for Bank**

In your `ApplicationRequestDto`, attach constraints to `BankGroup` (others can stay on `Default`).

```java
import jakarta.validation.constraints.*;
import org.hibernate.validator.group.GroupSequenceProvider;

@GroupSequenceProvider(ApplicationRequestDtoGroupSequenceProvider.class)
public class ApplicationRequestDto {

    @NotBlank
    private String applicationNo;

    @NotBlank
    private String channel; // <-- will drive the group selection

    // Example: these validate only when channel == Bank
    @NotBlank(groups = BankGroup.class)
    private String insuranceType;

    @NotBlank(groups = BankGroup.class)
    private String applicationType;

    // ...rest of your fields & nested DTOs; put groups = BankGroup.class where needed

    // getters/setters
}
```

### 3) Provide a **GroupSequenceProvider** that adds `BankGroup` only when channel == "Bank"

```java
import org.hibernate.validator.spi.group.DefaultGroupSequenceProvider;
import java.util.ArrayList;
import java.util.List;

public class ApplicationRequestDtoGroupSequenceProvider
        implements DefaultGroupSequenceProvider<ApplicationRequestDto> {

    @Override
    public List<Class<?>> getValidationGroups(ApplicationRequestDto dto) {
        List<Class<?>> groups = new ArrayList<>();
        groups.add(ApplicationRequestDto.class); // always include the bean class (Default group)

        if (dto != null && dto.getChannel() != null
                && "Bank".equalsIgnoreCase(dto.getChannel().trim())) {
            groups.add(BankGroup.class);
        }
        return groups;
    }
}
```

### 4) Controller stays the same (you can keep `@Valid`)

```java
@PostMapping("/submit")
public Response<String> saveApplication(@RequestBody @Valid ApplicationRequestDto applicationRequestDto) {
    applicationService.submitApplication(applicationRequestDto);
    return new Response.Builder<String>()
            .setStatus(1000)
            .setMessage("Success")
            .build();
}
```

**How it works:**

*   If `channel == "Bank"`, Hibernate Validator activates `BankGroup` → fields annotated with `groups = BankGroup.class` are validated.
*   Else, those constraints are ignored.

> This requires Hibernate Validator (the default in Spring Boot). No controller changes beyond `@Valid`.

***

## ✅ Option B — Programmatic validation in controller/service (simple & explicit)

If you prefer to **control it manually**, you can trigger validation only for the Bank case.

### 1) Keep your DTO constraints on a group

```java
public interface BankGroup {}
// annotate bank-only fields with groups = BankGroup.class in the DTO
```

### 2) Inject a `Validator` and validate conditionally

```java
import jakarta.validation.Validator;
import jakarta.validation.ConstraintViolation;
import org.springframework.http.ResponseEntity;

@RestController
@RequiredArgsConstructor
public class ApplicationController {

    private final Validator validator;
    private final ApplicationService applicationService;

    @PostMapping("/submit")
    public Response<String> saveApplication(@RequestBody ApplicationRequestDto dto) {

        if ("Bank".equalsIgnoreCase(dto.getChannel())) {
            var violations = validator.validate(dto, BankGroup.class);
            if (!violations.isEmpty()) {
                // turn into your error response shape
                String msg = violations.stream()
                        .map(v -> v.getPropertyPath() + " " + v.getMessage())
                        .reduce((a, b) -> a + "; " + b)
                        .orElse("Validation failed");
                throw new IllegalArgumentException(msg); // or build a proper 400 response
            }
        }
        // else: no validation

        applicationService.submitApplication(dto);

        return new Response.Builder<String>()
                .setStatus(1000)
                .setMessage("Success")
                .build();
    }
}
```

**How it works:**

*   When `channel == "Bank"` → it validates with `BankGroup`.
*   Otherwise → it skips validation.
*   This approach avoids dynamic group providers but requires manual error handling.

***

## Which should you use?

*   **Use Option A** if you want a **declarative** solution that plugs into `@Valid` seamlessly (cleaner controllers).
*   **Use Option B** if you want **explicit control** in the controller/service and minimal extra classes.

***

## Quick checklist

*   Add `groups = BankGroup.class` only to constraints that must run for Bank channel.
*   Keep other constraints on the default group if they must **always** run regardless of channel.
*   Be careful with nested DTOs: if you need conditional validation there too, annotate their constraints with `BankGroup` as well (or add their own group providers).

If you share which fields should validate **only** for Bank, I can mark those constraints precisely in your DTOs.


Here’s a **complete, minimal example for Option B (programmatic validation)**—you trigger validation **only when `channel == "Bank"`**.  
This keeps your controller in control and uses **validation groups** for Bank-only rules.

***

## 1) Define a validation group

```java
// src/main/java/com/example/validation/BankGroup.java
package com.example.validation;

public interface BankGroup {}
```

***

## 2) Annotate your DTO:

*   Put **Bank-only constraints** under `BankGroup`
*   Keep always-on constraints without a group (Default)

```java
// src/main/java/com/example/dto/ApplicationRequestDto.java
package com.example.dto;

import com.example.validation.BankGroup;
import jakarta.validation.Valid;
import jakarta.validation.constraints.*;

import java.util.List;

public class ApplicationRequestDto {

    @NotBlank // always validate
    private String applicationNo;

    @NotBlank // used to decide whether to validate with BankGroup
    private String channel;

    // Bank-only validations (run only when channel == "Bank")
    @NotBlank(groups = BankGroup.class)
    private String insuranceType;

    @NotBlank(groups = BankGroup.class)
    private String applicationType;

    // Example nested DTO
    @Valid
    private PolicyInsuredInfoDto policyInsuredInfo;

    // getters/setters ...
}
```

> Mark **only the fields you want to validate for Bank** with `groups = BankGroup.class`.

***

## 3) Controller: Validate conditionally using `Validator`

```java
// src/main/java/com/example/controller/ApplicationController.java
package com.example.controller;

import com.example.dto.ApplicationRequestDto;
import com.example.service.ApplicationService;
import com.example.validation.BankGroup;
import jakarta.validation.ConstraintViolation;
import jakarta.validation.Validator;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Set;
import java.util.stream.Collectors;

@RestController
@RequestMapping("/api/applications")
@RequiredArgsConstructor
public class ApplicationController {

    private final Validator validator;
    private final ApplicationService applicationService;

    @PostMapping("/submit")
    public ResponseEntity<?> saveApplication(@RequestBody ApplicationRequestDto dto) {

        // Always-on basic checks can be done here if needed:
        // var baseViolations = validator.validate(dto); // Default group
        // if (!baseViolations.isEmpty()) return badRequest(baseViolations);

        // Validate Bank-only rules conditionally
        if ("Bank".equalsIgnoreCase(dto.getChannel())) {
            Set<ConstraintViolation<ApplicationRequestDto>> bankViolations =
                    validator.validate(dto, BankGroup.class);

            if (!bankViolations.isEmpty()) {
                // Return a 400 with all validation messages
                var errors = bankViolations.stream()
                        .map(v -> v.getPropertyPath() + ": " + v.getMessage())
                        .collect(Collectors.toList());

                return ResponseEntity.badRequest().body(new ErrorResponse(400, "Validation failed (Bank)", errors));
            }
        }

        // Proceed if ok
        applicationService.submitApplication(dto);

        return ResponseEntity.ok(new ApiResponse<>(1000, "Success"));
    }

    // Simple response wrappers
    record ApiResponse<T>(int status, String message) {}
    record ErrorResponse(int status, String message, Object errors) {}
}
```

### Notes

*   We did **not** use `@Valid` on the parameter because we’re handling validation manually.  
    If you still want always-on default validation via `@Valid`, leave it and **also** run the group validation when `channel == "Bank"`.

***

## 4) Optional: Validate nested objects only for Bank

If you have Bank-only fields inside nested DTOs, annotate them with `groups = BankGroup.class` as well:

```java
// src/main/java/com/example/dto/PolicyInsuredInfoDto.java
package com.example.dto;

import com.example.validation.BankGroup;
import jakarta.validation.constraints.NotBlank;

public class PolicyInsuredInfoDto {

    @NotBlank(groups = BankGroup.class)
    private String nameInEnglish;

    // others...
}
```

The controller’s call `validator.validate(dto, BankGroup.class)` will cascade into nested DTOs annotated with `@Valid` on the parent (if you add it), or you can validate nested objects explicitly.

***

## 5) Service (unchanged)

Your service stays the same—no validation logic needed:

```java
// src/main/java/com/example/service/ApplicationService.java
package com.example.service;

import com.example.dto.ApplicationRequestDto;
import org.springframework.stereotype.Service;

@Service
public class ApplicationService {
    public void submitApplication(ApplicationRequestDto dto) {
        // ... your save flow
    }
}
```

***

## Variations you might want

*   **Also validate Default group** (always-on) before Bank group:
    ```java
    var defaults = validator.validate(dto); // Default group
    if (!defaults.isEmpty()) { ... return 400 ... }

    if ("Bank".equalsIgnoreCase(dto.getChannel())) {
        var bankOnly = validator.validate(dto, BankGroup.class);
        if (!bankOnly.isEmpty()) { ... return 400 ... }
    }
    ```

*   **Case-insensitive and trimmed channel** (already handled with `equalsIgnoreCase`).

***

### TL;DR

*   Define `BankGroup`
*   Annotate Bank-only constraints with `groups = BankGroup.class`
*   In controller: when `channel == "Bank"`, call `validator.validate(dto, BankGroup.class)` and return 400 if violations exist.

If you drop your current `ApplicationRequestDto` (or the fields you want Bank-only), I can mark the exact constraints with the proper group for you.


Here’s a **complete, minimal example for Option B (programmatic validation)**—you trigger validation **only when `channel == "Bank"`**.  
This keeps your controller in control and uses **validation groups** for Bank-only rules.

***

## 1) Define a validation group

```java
// src/main/java/com/example/validation/BankGroup.java
package com.example.validation;

public interface BankGroup {}
```

***

## 2) Annotate your DTO:

*   Put **Bank-only constraints** under `BankGroup`
*   Keep always-on constraints without a group (Default)

```java
// src/main/java/com/example/dto/ApplicationRequestDto.java
package com.example.dto;

import com.example.validation.BankGroup;
import jakarta.validation.Valid;
import jakarta.validation.constraints.*;

import java.util.List;

public class ApplicationRequestDto {

    @NotBlank // always validate
    private String applicationNo;

    @NotBlank // used to decide whether to validate with BankGroup
    private String channel;

    // Bank-only validations (run only when channel == "Bank")
    @NotBlank(groups = BankGroup.class)
    private String insuranceType;

    @NotBlank(groups = BankGroup.class)
    private String applicationType;

    // Example nested DTO
    @Valid
    private PolicyInsuredInfoDto policyInsuredInfo;

    // getters/setters ...
}
```

> Mark **only the fields you want to validate for Bank** with `groups = BankGroup.class`.

***

## 3) Controller: Validate conditionally using `Validator`

```java
// src/main/java/com/example/controller/ApplicationController.java
package com.example.controller;

import com.example.dto.ApplicationRequestDto;
import com.example.service.ApplicationService;
import com.example.validation.BankGroup;
import jakarta.validation.ConstraintViolation;
import jakarta.validation.Validator;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Set;
import java.util.stream.Collectors;

@RestController
@RequestMapping("/api/applications")
@RequiredArgsConstructor
public class ApplicationController {

    private final Validator validator;
    private final ApplicationService applicationService;

    @PostMapping("/submit")
    public ResponseEntity<?> saveApplication(@RequestBody ApplicationRequestDto dto) {

        // Always-on basic checks can be done here if needed:
        // var baseViolations = validator.validate(dto); // Default group
        // if (!baseViolations.isEmpty()) return badRequest(baseViolations);

        // Validate Bank-only rules conditionally
        if ("Bank".equalsIgnoreCase(dto.getChannel())) {
            Set<ConstraintViolation<ApplicationRequestDto>> bankViolations =
                    validator.validate(dto, BankGroup.class);

            if (!bankViolations.isEmpty()) {
                // Return a 400 with all validation messages
                var errors = bankViolations.stream()
                        .map(v -> v.getPropertyPath() + ": " + v.getMessage())
                        .collect(Collectors.toList());

                return ResponseEntity.badRequest().body(new ErrorResponse(400, "Validation failed (Bank)", errors));
            }
        }

        // Proceed if ok
        applicationService.submitApplication(dto);

        return ResponseEntity.ok(new ApiResponse<>(1000, "Success"));
    }

    // Simple response wrappers
    record ApiResponse<T>(int status, String message) {}
    record ErrorResponse(int status, String message, Object errors) {}
}
```

### Notes

*   We did **not** use `@Valid` on the parameter because we’re handling validation manually.  
    If you still want always-on default validation via `@Valid`, leave it and **also** run the group validation when `channel == "Bank"`.

***

## 4) Optional: Validate nested objects only for Bank

If you have Bank-only fields inside nested DTOs, annotate them with `groups = BankGroup.class` as well:

```java
// src/main/java/com/example/dto/PolicyInsuredInfoDto.java
package com.example.dto;

import com.example.validation.BankGroup;
import jakarta.validation.constraints.NotBlank;

public class PolicyInsuredInfoDto {

    @NotBlank(groups = BankGroup.class)
    private String nameInEnglish;

    // others...
}
```

The controller’s call `validator.validate(dto, BankGroup.class)` will cascade into nested DTOs annotated with `@Valid` on the parent (if you add it), or you can validate nested objects explicitly.

***

## 5) Service (unchanged)

Your service stays the same—no validation logic needed:

```java
// src/main/java/com/example/service/ApplicationService.java
package com.example.service;

import com.example.dto.ApplicationRequestDto;
import org.springframework.stereotype.Service;

@Service
public class ApplicationService {
    public void submitApplication(ApplicationRequestDto dto) {
        // ... your save flow
    }
}
```

***

## Variations you might want

*   **Also validate Default group** (always-on) before Bank group:
    ```java
    var defaults = validator.validate(dto); // Default group
    if (!defaults.isEmpty()) { ... return 400 ... }

    if ("Bank".equalsIgnoreCase(dto.getChannel())) {
        var bankOnly = validator.validate(dto, BankGroup.class);
        if (!bankOnly.isEmpty()) { ... return 400 ... }
    }
    ```

*   **Case-insensitive and trimmed channel** (already handled with `equalsIgnoreCase`).

***

### TL;DR

*   Define `BankGroup`
*   Annotate Bank-only constraints with `groups = BankGroup.class`
*   In controller: when `channel == "Bank"`, call `validator.validate(dto, BankGroup.class)` and return 400 if violations exist.

If you drop your current `ApplicationRequestDto` (or the fields you want Bank-only), I can mark the exact constraints with the proper group for you.


Below is a **controller in your style** that conditionally validates the DTO:

*   If `channel == "Bank"` → **run validation** (Bank-only rules) and proceed if OK.
*   If `channel == "GSSP"` → **skip validation** and proceed.
*   Otherwise → you can choose to validate or skip (shown as skip in this example; tweak as you need).

> Assumptions:
>
> *   You have a marker interface `BankGroup` (empty interface).
> *   Your custom `Response` builder is available as in your snippet.
> *   You’re fine with returning a `400`-style response using your `Response` wrapper when validation fails.

```java
package com.example.controller;

import com.example.dto.ApplicationRequestDto;
import com.example.service.ApplicationService;
import com.example.validation.BankGroup;
import jakarta.validation.ConstraintViolation;
import jakarta.validation.Validator;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Set;
import java.util.stream.Collectors;

@RestController
@RequestMapping("/api/applications")
@RequiredArgsConstructor
public class ApplicationController {

    private final ApplicationService applicationService;
    private final Validator validator;

    @PostMapping("/submit")
    public Response<String> saveApplication(@RequestBody ApplicationRequestDto applicationRequestDto) {

        // Conditionally validate based on channel
        String channel = applicationRequestDto.getChannel();

        if ("Bank".equalsIgnoreCase(channel)) {
            // Validate ONLY Bank-group rules
            Set<ConstraintViolation<ApplicationRequestDto>> violations =
                    validator.validate(applicationRequestDto, BankGroup.class);

            if (!violations.isEmpty()) {
                // Build a single message or join multiple messages as you prefer
                String message = violations.stream()
                        .map(v -> v.getPropertyPath() + ": " + v.getMessage())
                        .collect(Collectors.joining("; "));

                return new Response.Builder<String>()
                        .setStatus(1400)
                        .setMessage(message)
                        .build();
            }
        } else if ("GSSP".equalsIgnoreCase(channel)) {
            // Skip validation entirely and proceed
        } else {
            // For other channels you can decide to validate or skip.
            // Currently skipping; uncomment below if you want default validation:
            // Set<ConstraintViolation<ApplicationRequestDto>> defaultViolations = validator.validate(applicationRequestDto);
            // if (!defaultViolations.isEmpty()) { ... return error ... }
        }

        // Proceed to service
        applicationService.submitApplication(applicationRequestDto);

        return new Response.Builder<String>()
                .setStatus(1000)
                .setMessage("Success")
                .build();
    }
}
```

### Marker group (for completeness)

```java
package com.example.validation;

public interface BankGroup {}
```

***

### Notes

*   Put **Bank-only constraints** in your DTOs with `groups = BankGroup.class`, e.g.:
    ```java
    @NotBlank(groups = BankGroup.class)
    private String insuranceType;
    ```
*   Keep fields that must **always** validate (regardless of channel) **without** a group (Default), and then call `validator.validate(dto)` before the `if` if you want those to run. In your requirement, you said **no validation** for `GSSP`, so I didn’t add default validation.
*   If you prefer `ResponseEntity<Response<String>>`, you can wrap the success/error responses with `ResponseEntity.ok(...)` or `ResponseEntity.badRequest().body(...)`. I kept your exact return type.

If you want me to wire **nested DTO validations** to only apply for `Bank` (e.g., inside `policyInsuredInfo`), share those DTOs and I’ll add the `groups = BankGroup.class` where needed.
