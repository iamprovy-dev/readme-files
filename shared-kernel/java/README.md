# shared-kernel

Uniform API responses, robust validation, and shared business utilities for Spring Boot. Available from Maven Central.

## Overview

This library provides a complete solution for building consistent RESTful APIs with:

- **Standardized API Responses** - Consistent JSON envelope format across your application
- **Validation Utilities** - Comprehensive field validation helpers
- **Custom Exceptions** - Pre-defined exception types for common scenarios
- **Global Exception Handling** - Automatic HTTP status code mapping
- **Custom Validators** - Jakarta Bean Validation annotations
- **Utility Classes** - BigDecimal operations and more

## Installation

**Repository:** [com.syncafricabs/shared-kernel](https://central.sonatype.com/artifact/com.syncafricabs/shared-kernel)

Add this dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>com.syncafricabs</groupId>
    <artifactId>shared-kernel</artifactId>
    <version>1.0.0-beta</version>
</dependency>
```

## API Response Format

All responses follow this standardized JSON envelope:

```json
{
  "code": 200,
  "message": "Operation successful",
  "success": true,
  "data": { ... }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `code` | `number` | HTTP status code (200, 201, 400, 401, 403, 404, 409, 500, etc.) |
| `message` | `string` | Human-readable message |
| `success` | `boolean` | `true` for successful operations, `false` for failures |
| `data` | `any` | Response payload (null for error responses) |

## Quick Start

### Build API Responses

```java
import com.syncafricabs.shared_kernel.ApiEnvelopeFactory;
import com.syncafricabs.shared_kernel.ResponseEntityBuilder;
import com.syncafricabs.shared_kernel.ApiEnvelope;

// Create a success response
ApiEnvelope<User> response = ApiEnvelopeFactory.success("User retrieved", user);

// Or return directly as ResponseEntity
return ResponseEntityBuilder.ok("User retrieved", user);
return ResponseEntityBuilder.created("User created", user);
return ResponseEntityBuilder.notFound("User not found");
return ResponseEntityBuilder.badRequest("Invalid input");
return ResponseEntityBuilder.internalServerError("Something went wrong");
```

### Validate Input Fields

```java
import com.syncafricabs.shared_kernel.validation.GlobalFieldValidator;

// Validate strings
GlobalFieldValidator.validateString(name, "Name", 2, 100);

// Validate email
GlobalFieldValidator.validateEmail(email, "Email");

// Validate phone
GlobalFieldValidator.validatePhone(phone, "Phone");

// Validate URL
GlobalFieldValidator.validateUrl(website, "Website");

// Validate UUID
GlobalFieldValidator.validateUUID(id, "Id");

// Validate positive Long ID
GlobalFieldValidator.validatePositiveLong(userId, "UserId");

// Validate positive decimal amount
GlobalFieldValidator.validatePositiveAmount(amount, "Amount");

// Validate percentage (0-100)
GlobalFieldValidator.validatePercentage(rate, "Rate");

// Validate non-null value
GlobalFieldValidator.validateNonNull(value, "Value");

// Validate non-empty list
GlobalFieldValidator.validateNonEmptyList(items, "Items");

// Validate enum value
GlobalFieldValidator.validateEnum(Gender.class, gender, "Gender");

// Validate date of birth (at least 2 years old)
GlobalFieldValidator.validateDateOfBirth(dob, "DateOfBirth");

// Validate payment date (not in future)
GlobalFieldValidator.validatePaymentDate(paymentDate, "PaymentDate");

// Validate opening date (not more than 7 days in past)
GlobalFieldValidator.validateOpeningDate(openingDate, "OpeningDate");

// Validate integer
GlobalFieldValidator.validateInteger(count, "Count");

// Validate period (days, 0-365)
GlobalFieldValidator.validatePeriod(days, "Days");

// Validate size limit (1-1,000,000)
GlobalFieldValidator.validateSizeLimit(limit, "Limit");

// Validate fee (non-negative, max 2 decimal places)
GlobalFieldValidator.validateFee(fee, "Fee");
```

### Handle Exceptions

The library includes a `@RestControllerAdvice` that automatically maps exceptions to standardized API responses:

```java
import com.syncafricabs.shared_kernel.exception.*;

// Throw these in your service layer - they are automatically handled
throw new ValidationException("Invalid input");
throw new MissingFieldsException("Email is required");
throw new AlreadyExistsException("User already exists");
throw new ResourceNotFoundException("User not found");
throw new UnauthorizedException("Invalid credentials");
throw new NotAllowedException("Access denied");
throw new InvalidException("Invalid state");
throw new RequestFailedException("Operation failed");
throw new InsufficientFundsException("Insufficient balance");
```

**Exception-to-HTTP Mapping:**

| Exception | HTTP Status | Use Case |
|-----------|-------------|----------|
| `ValidationException` | 400 | Validation errors, invalid input |
| `MissingFieldsException` | 400 | Required fields missing |
| `AlreadyExistsException` | 409 | Resource already exists |
| `ResourceNotFoundException` | 404 | Resource not found |
| `UnauthorizedException` | 401 | Authentication required |
| `NotAllowedException` | 403 | Access denied |
| `InvalidException` | 400 | Invalid state or value |
| `RequestFailedException` | 500 | Server-side request failure |
| `InsufficientFundsException` | 400 | Insufficient funds |
| Generic `Exception` | 500 | Unhandled server errors |

### Custom Jakarta Validators

```java
import com.syncafricabs.shared_kernel.validation.ValidUuid;
import com.syncafricabs.shared_kernel.validation.ValidEnum;
import com.syncafricabs.shared_kernel.validation.ValidLongRange;

public class UserDto {

    @ValidUuid
    private String id;

    @ValidEnum(enumClass = Gender.class)
    private String gender;

    @ValidLongRange(min = 1, max = 1000000)
    private Long userId;
}
```

### BigDecimal Utilities

```java
import com.syncafricabs.shared_kernel.configs.BigDecimalUtils;

// Null-safe operations
BigDecimal result = BigDecimalUtils.add(price, tax);
BigDecimal total = BigDecimalUtils.multiply(quantity, unitPrice);

// Percentage calculations
BigDecimal tax = BigDecimalUtils.percentageOf(amount, BigDecimal.valueOf(16));
BigDecimal discounted = BigDecimalUtils.applyDiscount(price, BigDecimal.valueOf(10));

// Comparison
boolean isGreater = BigDecimalUtils.isGreaterThan(a, b);
BigDecimal max = BigDecimalUtils.max(a, b);

// Formatting
String formatted = BigDecimalUtils.formatAsCurrency(amount);
String percentage = BigDecimalUtils.formatAsPercentage(rate);
```

### Controller Example

```java
import com.syncafricabs.shared_kernel.ResponseEntityBuilder;
import com.syncafricabs.shared_kernel.ApiEnvelope;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public ResponseEntity<ApiEnvelope<User>> createUser(@Valid @RequestBody UserDto dto) {
        User user = userService.create(dto);
        return ResponseEntityBuilder.created("User created successfully", user);
    }

    @GetMapping("/{id}")
    public ResponseEntity<ApiEnvelope<User>> getUser(@PathVariable Long id) {
        GlobalFieldValidator.validatePositiveLong(id, "Id");
        User user = userService.findByIdOrThrow(id);
        return ResponseEntityBuilder.ok("User retrieved", user);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<ApiEnvelope<Void>> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntityBuilder.noContent("User deleted");
    }
}
```

## Project Structure

## License

Apache License, Version 2.0

## Author

**Providence Chikukwa**
- Email: iamprovy@outlook.com
- GitHub: https://github.com/iamprovy-dev
- LinkedIn: https://www.linkedin.com/in/provychikukwa
- Organization: SyncAfrica Business Solutions (https://www.syncafricabs.com)
