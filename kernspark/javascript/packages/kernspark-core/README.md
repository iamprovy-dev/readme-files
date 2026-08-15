# @syncafricabs/kernspark-core

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue)](https://www.typescriptlang.org/)

Framework-independent core for standardized API responses, validation, result types, and domain primitives.

## Table of Contents

- [What is it?](#what-is-it)
- [Why does it exist?](#why-does-it-exist)
- [Architecture](#architecture)
- [Packages](#packages)
- [When to use it](#when-to-use-it)
- [When NOT to use it](#when-not-to-use-it)
- [Installation](#installation)
- [Core Concepts](#core-concepts)
    - [Data Envelope](#data-envelope)
    - [Application Errors](#application-errors)
    - [Validation](#validation)
    - [Utilities](#utilities)
- [Usage Example](#usage-example)
    - [Object-Level Validation Pattern](#object-level-validation-pattern)
    - [Per-Field Validation Pattern (Recommended)](#per-field-validation-pattern-recommended)
- [API Reference](#api-reference)
- [Error Handling Reference](#error-handling-reference)
- [Compatibility Matrix](#compatibility-matrix)
- [Contributing](#contributing)
- [License](#license)
- [Author](#author)

## What is it?

`@syncafricabs/kernspark-core` provides framework-independent building blocks for building consistent, type-safe TypeScript/JavaScript applications:

- **Data Envelope** — Standardized `DataEnvelope<T>` response wrapper for consistent success/error payloads
- **Application Errors** — Refined exception hierarchy with error codes, HTTP status codes, and cause chains
- **Validation** — Generic field validators that complement Zod, Joi, or class-validator
- **Utilities** — Generic helpers for common operations

The library has **zero runtime dependencies**. It can be used in any TypeScript/JavaScript project.

## Why does it exist?

Every backend team eventually builds the same primitives:

- Standardized JSON response envelopes
- Exception hierarchies
- Input validation helpers
- Global error handlers

Instead of each team reinventing these, the core provides a **single, well-tested, open-source foundation** that works across frameworks.

## Architecture

```
@syncafricabs/kernspark-core
    │
    ├── Framework-independent functionality
    │       ├── DataEnvelope
    │       ├── DataEnvelopeFactory
    │       ├── Application/domain exceptions
    │       ├── Validation primitives
    │       ├── Result<T, E>
    │       ├── Pagination
    │       ├── Common types
    │       └── Generic utilities
    │
    └── Framework adapters (separate packages)
            ├── @syncafricabs/kernspark-express
            ├── @syncafricabs/kernspark-nestjs
            ├── @syncafricabs/kernspark-fastify
            └── ...
```

**Key architectural rule:** The core package must **NOT** depend on Express, NestJS, Fastify, Hono, or any specific web framework. Framework-specific functionality belongs in adapter packages.

## Packages

| Package | Description |
|---------|-------------|
| `@syncafricabs/kernspark-core` | Framework-independent core |
| `@syncafricabs/kernspark-express` | Express adapter |
| `@syncafricabs/kernspark-nestjs` | NestJS adapter |
| `@syncafricabs/kernspark-fastify` | Fastify adapter |
| `@syncafricabs/kernspark-hono` | Hono adapter |
| `@syncafricabs/kernspark-nextjs` | Next.js adapter |
| `@syncafricabs/kernspark-nuxt` | Nuxt adapter |
| `@syncafricabs/kernspark-sveltekit` | SvelteKit adapter |
| `@syncafricabs/kernspark-remix` | Remix adapter |
| `@syncafricabs/kernspark-elysiajs` | ElysiaJS adapter |
| `@syncafricabs/kernspark-koa` | Koa adapter |
| `@syncafricabs/kernspark-nitro` | Nitro adapter |
| `@syncafricabs/kernspark-adonisjs` | AdonisJS adapter |
| `@syncafricabs/kernspark-feathers` | Feathers adapter |
| `@syncafricabs/kernspark-sailsjs` | SailsJS adapter |
| `@syncafricabs/kernspark-strapi` | Strapi adapter |
| `@syncafricabs/kernspark-trpc` | tRPC adapter |

## When to use it

- You want **consistent API responses** across multiple services or frameworks
- You need a **shared exception hierarchy** that works across bounded contexts
- You want to reduce boilerplate for validation, pagination, and response formatting
- You are building a **microservice architecture** where multiple teams share common contracts
- You want **framework flexibility** — core logic that can run on Express, NestJS, Fastify, Hono, Next.js, Nuxt, SvelteKit, Remix, ElysiaJS, Koa, Nitro, AdonisJS, Feathers, SailsJS, Strapi, tRPC, or any other supported framework
- You need **production-ready** building blocks that are open-source and community-driven

## When NOT to use it

- You need a full backend framework (this is not Express, NestJS, Fastify, Hono, Next.js, Nuxt, SvelteKit, Remix, ElysiaJS, Koa, Nitro, AdonisJS, Feathers, SailsJS, Strapi, or tRPC)
- You need an ORM or database access layer (use Prisma, TypeORM, etc.)
- You need authentication/authorization frameworks (use Passport, Casbin, etc.)
- You need logging frameworks (use Pino, Winston, etc.)
- You need validation frameworks (use Zod, Joi, class-validator — this complements them)
- You need application-specific business logic (keep that in your domain layer)

**Philosophy:** Complement the existing ecosystem, don't replace it.

## Installation

### Core only

If you only want framework-independent functionality:

```bash
npm install @syncafricabs/kernspark-core
```

## Core Concepts

### Data Envelope

`DataEnvelope<T>` is a plain class (not a sealed interface with `Delivered`/`Failed` variants). Every API response — success or error — is returned as this same shape:

```typescript
import { DataEnvelopeFactory } from '@syncafricabs/kernspark-core';

// Success
const response = DataEnvelopeFactory.success('OK', { id: 1, name: 'John' });

// Error
const error = DataEnvelopeFactory.notFound('USER_NOT_FOUND', 'User not found', null);

// Same type either way
response.success;     // true for success, false for error
response.code;       // HTTP status code
response.data;       // payload, or null for errors
response.message;    // human-readable description ("ERROR_CODE: message" for errors)
```

On the wire it serializes as:

```json
// Success
{
  "code": 200,
  "success": true,
  "data": { "id": 1, "name": "John" },
  "message": "OK"
}

// Error
{
  "code": 400,
  "success": false,
  "message": "USER_EMAIL_REQUIRED: Email is required",
  "data": null
}
```

**Key design decisions:**

- `code` is the HTTP status code
- `success` is a boolean discriminator for type-safe handling
- `data` carries the payload for success responses (or `null` for errors)
- `message` is the human-readable description

**No separate `errorCode` field.** `DataEnvelope` intentionally stays flat with just `code` / `message` / `success` / `data`. `DataEnvelopeFactory`'s error-producing methods (`badRequest`, `notFound`, `conflict`, etc.) still accept an `errorCode` parameter, but it's folded directly into `message` as `"ERROR_CODE: human readable message"` rather than shipped as its own JSON field:

```json
{
  "code": 400,
  "success": false,
  "message": "USER_EMAIL_REQUIRED: Email is required",
  "data": null
}
```

If you need `errorCode` to stay machine-parseable as its own field on the client side, split on the first `": "` in `message`, or wrap `DataEnvelope` with your own DTO before it leaves the API layer.

### Application Errors

The library provides a refined exception hierarchy:

```
ApplicationError (base)
├── ValidationError
│   ├── MissingFieldsError
│   └── InvalidError
├── BusinessError
│   ├── AlreadyExistsError
│   ├── InsufficientFundsError
│   ├── QuotaExceededError
│   ├── NotFoundError
│   ├── ExpiredError
│   ├── TooManyRequestsError
│   ├── PaymentRequiredError
│   ├── LockedError
│   ├── AccountSuspendedError
│   ├── FeatureNotAvailableError
│   └── DataIntegrityError
├── AuthenticationError
│   ├── InvalidTokenError
│   ├── TokenExpiredError
│   └── SessionExpiredError
├── AuthorizationError
│   ├── PermissionDeniedError
│   └── NotAllowedError
└── InfrastructureError
    ├── ExternalServiceError
    ├── BadGatewayError
    ├── GatewayTimeoutError
    ├── ServiceUnavailableError
    ├── RequestFailedError
    ├── NotImplementedError
    └── MaintenanceModeError
```

Each exception carries:
- `errorCode` — machine-readable code (e.g., `USER_NOT_FOUND`)
- `statusCode` — HTTP status for adapters
- `message` — human-readable description
- `cause` — optional underlying error

**Usage:**

```typescript
import { NotFoundError, ValidationError, AlreadyExistsError } from '@syncafricabs/kernspark-core';

// Throw in your service layer
throw new NotFoundError('User not found');
throw new ValidationError('INVALID_EMAIL', 'Email format is invalid');
throw new AlreadyExistsError('User already exists');
```

The framework adapters automatically map these to HTTP responses.

### Validation

`FieldValidator` provides generic validation primitives that complement Zod, Joi, or class-validator:

```typescript
import { FieldValidator } from '@syncafricabs/kernspark-core';

// Strings
FieldValidator.validateString(name, 'Name', 2, 100);

// Email
FieldValidator.validateEmail(email, 'Email');

// Phone
FieldValidator.validatePhone(phone, 'Phone');

// URL
FieldValidator.validateUrl(website, 'Website');

// UUID
FieldValidator.validateUUID(id, 'Id');

// Numbers
FieldValidator.validatePositiveLong(userId, 'UserId');
FieldValidator.validateInteger(count, 'Count');
FieldValidator.validatePeriod(days, 'Days');
FieldValidator.validateSizeLimit(limit, 'Limit');
FieldValidator.validateFee(fee, 'Fee');

// Other
FieldValidator.validateNonNull(value, 'Value');
FieldValidator.validateNonEmptyList(items, 'Items');
FieldValidator.validateEnum(Gender, gender, 'Gender');
FieldValidator.validateBoolean(active, 'Active');
FieldValidator.validatePattern(code, /^[A-Z]{3}$/, 'Code');
FieldValidator.validateNumericRange(age, 18, 120, 'Age');
FieldValidator.validateColumnName(columnName, 'ColumnName');
FieldValidator.validateDateOfBirth(dob, 'DateOfBirth');
FieldValidator.validatePaymentDate(paymentDate, 'PaymentDate');
FieldValidator.validateOpeningDate(openingDate, 'OpeningDate');
```

**Important:** Business rules should remain in your application's domain/application layer. This package provides generic primitives, not a complete validation framework.

### Utilities

`BaseUtils` provides null-safe helpers for common operations:

```typescript
import { BaseUtils } from '@syncafricabs/kernspark-core';

BaseUtils.isNotNullOrUndefined(value);  // boolean
BaseUtils.isNullOrUndefined(value);     // boolean
BaseUtils.isEmptyString(value);         // boolean
BaseUtils.isNotEmptyString(value);       // boolean
BaseUtils.sleep(1000);                   // Promise<void>
BaseUtils.randomString(16);              // random alphanumeric string
```

**Note:** HTML sanitization has been removed from the core. Use a dedicated, well-maintained sanitization library for security-sensitive operations.

## Usage Example

Kernspark supports two common patterns for wiring `FieldValidator` into a validation flow, depending on whether you want a single flat validation message or a field-keyed error map. Both use the same request/response models and the same `DataEnvelopeFactory` — the only difference is how validation failures are reported.

### Model — Request

The request model is a plain data holder. It carries no validation or HTTP logic of its own.

```typescript
interface SampleClassRequest {
  fullName: string;
  phoneNumber: string;
  emailAddress: string;
  uuidValue: string;
  amount: number;
  paymentDate: Date;
  gender: 'MALE' | 'FEMALE' | 'OTHER';
}
```

### Model — Response

The response model represents the data returned after the request has been successfully processed.

```typescript
interface SampleClassResponse {
  fullName: string;
  phoneNumber: string;
  emailAddress: string;
  uuidValue: string;
  amount: number;
  paymentDate: Date;
  gender: 'MALE' | 'FEMALE' | 'OTHER';
}
```

### Service

`SampleService` doesn't need to know which validation pattern the controller uses — by the time `createSample(...)` runs, the validator has already guaranteed the request is valid. The service goes straight to building the response, and still owns building the `DataEnvelope` via `DataEnvelopeFactory`, so the controller can return it unchanged. A `catch (Exception e)` remains as a safety net for unexpected runtime errors during processing; anything not caught here still falls through to the framework adapter's error handler.

```typescript
import { DataEnvelopeFactory, NotFoundError } from '@syncafricabs/kernspark-core';

export class SampleService {

  createSample(request: SampleClassRequest): DataEnvelope<SampleClassResponse> {

    try {
      const response: SampleClassResponse = {
        fullName: request.fullName,
        phoneNumber: request.phoneNumber,
        emailAddress: request.emailAddress,
        uuidValue: request.uuidValue,
        amount: request.amount,
        paymentDate: request.paymentDate,
        gender: request.gender,
      };

      console.log('Sample created for id:', request.uuidValue);

      return DataEnvelopeFactory.created(
        'Sample data passed the validation successfully',
        response
      );

    } catch (e) {
      console.error('Error creating sample data:', e);
      return DataEnvelopeFactory.internalServerError(
        'REQUEST_FAILED',
        'Failed to create sample data',
        null
      );
    }
  }
}
```

> **Note:** This `catch (e)` is optional — the framework adapter's error handler already produces an equivalent 500 response for any uncaught exception. Keep it only if you want service-specific logging or a custom message for this endpoint; otherwise you can drop the try/catch entirely and let unexpected errors propagate to the global handler.

### Object-Level Validation Pattern

This is the simplest pattern: each `FieldValidator` call runs in sequence, and the **first** `ValidationError` thrown stops the method. It's easy to write, but it means only the first invalid field is reported per request — a client fixing one field at a time will discover the next error only on their next submission.

```typescript
import { ValidationError, FieldValidator } from '@syncafricabs/kernspark-core';

export class SampleRequestValidator {

  static validate(request: SampleClassRequest): void {
    try {
      FieldValidator.validateString(request.fullName, 'User full name', 3, 30);
      FieldValidator.validateEmail(request.emailAddress, 'User email address');
      FieldValidator.validatePhone(request.phoneNumber, 'User phone number');
      FieldValidator.validateUUID(request.uuidValue, 'UUID value');
      FieldValidator.validateEnum(Gender, request.gender, 'User gender');
      FieldValidator.validatePaymentDate(request.paymentDate, 'Payment date');
    } catch (e) {
      if (e instanceof ValidationError) {
        throw e;
      }
      throw new ValidationError('VALIDATION_ERROR', 'Validation failed', e as Error);
    }
  }
}
```

### Controller

The controller calls `SampleRequestValidator.validate(...)` before passing the request to the service. If validation fails, the `ValidationError` is caught by the framework adapter's error handler and returned as a standardized error response. Since `SampleService.createSample(...)` already returns a fully built `DataEnvelope`, the controller just returns it directly.

```typescript
import { DataEnvelope } from '@syncafricabs/kernspark-core';

export class SampleController {

  static create(request: SampleClassRequest): DataEnvelope<any> {
    console.log('Creating sample:', request.fullName);

    SampleRequestValidator.validate(request);

    return SampleService.createSample(request);
  }
}
```

### Per-Field Validation Pattern (Recommended)

Instead of stopping at the first failure, this pattern runs **every** `FieldValidator` check independently, catching each `ValidationError` where it happens. The result is a map with one invalid field at a time, so the error handler can return **all** invalid fields in a single response — which is what a form-driven or mobile client typically wants, since it can highlight every invalid input at once instead of round-tripping one error at a time.

The trick is the small `checkField` helper: it wraps a single check in a try/catch so that one failing field never stops the remaining fields from also being checked.

```typescript
import { ValidationError, FieldValidator } from '@syncafricabs/kernspark-core';

export class SampleRequestValidator {

  static validate(request: SampleClassRequest): Record<string, string> {
    const errors: Record<string, string> = {};

    checkField(errors, 'fullName', () =>
      FieldValidator.validateString(request.fullName, 'User full name', 3, 30));

    checkField(errors, 'emailAddress', () =>
      FieldValidator.validateEmail(request.emailAddress, 'User email address'));

    checkField(errors, 'phoneNumber', () =>
      FieldValidator.validatePhone(request.phoneNumber, 'User phone number'));

    checkField(errors, 'uuidValue', () =>
      FieldValidator.validateUUID(request.uuidValue, 'UUID value'));

    checkField(errors, 'gender', () =>
      FieldValidator.validateEnum(Gender, request.gender, 'User gender'));

    checkField(errors, 'paymentDate', () =>
      FieldValidator.validatePaymentDate(request.paymentDate, 'Payment date'));

    if (Object.keys(errors).length > 0) {
      throw new ValidationError('VALIDATION_ERROR', 'Validation failed', JSON.stringify(errors));
    }

    return errors;
  }

  private static checkField(
    errors: Record<string, string>,
    fieldName: string,
    check: () => void
  ): void {
    try {
      check();
    } catch (e) {
      if (e instanceof ValidationError) {
        errors[fieldName] = e.message;
      } else {
        errors[fieldName] = 'Validation failed';
      }
    }
  }
}
```

#### Why report every invalid field at once?

With the object-level pattern, `FieldValidator` calls run in sequence and the first `ValidationError` thrown aborts the rest of the method — so a client only ever learns about one bad field per submission. With the per-field pattern, every check runs regardless of whether an earlier one failed, so a single submit can surface the complete picture:

- **Fewer round trips.** A frontend form with several invalid fields gets all the errors in one response, instead of the user fixing one field, resubmitting, hitting the next error, fixing that, resubmitting again, and so on.
- **Field-level highlighting.** Because each error is keyed by field name (`emailAddress`, `phoneNumber`, `uuidValue`, ...) rather than a single flat message, the frontend can map each entry in `data` directly onto the corresponding input and show an inline error next to it — no string-matching or guessing which field a generic message refers to.
- **Better perceived performance and lower frustration.** Users generally expect a "submit" click to validate the whole form, not just the first field top-to-bottom; discovering errors one at a time feels slow and unpredictable, especially on longer forms.
- **Fewer wasted requests.** Each avoidable resubmission is a full HTTP round trip through validation, binding, and the error handler for information the server already had on the first attempt — reporting everything up front cuts that down to as few requests as the data actually requires.

The trade-off is minor: every `FieldValidator` check always runs, even after an earlier one has already failed, so validation does slightly more work per request than the fail-fast object-level pattern. For typical request sizes this is negligible compared to the UX gain.

#### Example Request — Multiple Invalid Fields

```json
POST /api/samples/create
{
  "fullName": "Providence Chikukwa",
  "phoneNumber": "string",
  "emailAddress": "string",
  "uuidValue": "string",
  "amount": 5,
  "paymentDate": "2026-08-13",
  "gender": "MALE"
}
```

Because each check runs in its own isolated `checkField(...)` call, every invalid field is reported in one response — not just the first one encountered:

```json
{
  "code": 400,
  "message": "VALIDATION_ERROR: Validation failed",
  "success": false,
  "data": {
    "amount": "Amount must be at least 10",
    "emailAddress": "User email address has invalid format",
    "phoneNumber": "User phone number must be at least 10 characters long",
    "uuidValue": "Invalid UUID value value: "
  }
}
```

Note `fullName`, `gender`, and `paymentDate` passed validation and are correctly omitted from `data` — only the four invalid fields are reported.

#### Example Request — Valid Payload

```json
POST /api/samples/create
{
  "fullName": "Providence Chikukwa",
  "phoneNumber": "+263784340477",
  "emailAddress": "iamprovy@outlook.com",
  "uuidValue": "550e8400-e29b-41d4-a716-446655440000",
  "amount": 10000,
  "paymentDate": "2026-08-13",
  "gender": "MALE"
}
```

```json
{
  "code": 201,
  "message": "Sample data passed the validation successfully",
  "success": true,
  "data": {
    "amount": 10000,
    "emailAddress": "iamprovy@outlook.com",
    "fullName": "Providence Chikukwa",
    "gender": "MALE",
    "paymentDate": "2026-08-13",
    "phoneNumber": "+263784340477",
    "uuidValue": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### Validation Flow

The recommended flow, applicable to either pattern above, is:

1. The controller receives a `SampleClassRequest`.
2. The controller calls `SampleRequestValidator.validate(request)` before passing the request to the service.
3. The validator runs `FieldValidator` checks and reports failures — either as a single object-level error (object-level pattern), or as one error per invalid field (per-field pattern).
4. If validation fails, the `ValidationError` is caught by the framework adapter's error handler and returned as a standardized `DataEnvelope` bad-request response — the controller method never reaches the service.
5. If validation succeeds, the controller method runs with a request guaranteed to be valid.
6. `SampleService.createSample(...)` processes the request and returns a fully built `DataEnvelope` via `DataEnvelopeFactory`.
7. The controller returns the service's response directly.

This keeps validation and business logic in separate, single-responsibility places — the `Validator` and the `Service` — while the `Service` still owns response construction for this endpoint, and the framework adapter's error handler remains the fallback for anything the controller and service don't handle themselves.

## API Reference

### DataEnvelope

`DataEnvelope<T>` is a single concrete class (not an interface with separate success/error implementations) used as the return type for both outcomes:

```typescript
// Success response
DataEnvelope<T> response = DataEnvelopeFactory.success('OK', data);

// Error response
DataEnvelope<E> response = DataEnvelopeFactory.notFound('USER_NOT_FOUND', 'User not found', null);

// Same type either way
response.success;  // true for success, false for error
response.code;     // HTTP status code
response.data;     // payload, or null for errors
response.message;  // human-readable description ("ERROR_CODE: message" for errors)
```

### DataEnvelopeFactory

All methods return a plain `DataEnvelope<T>` (or `DataEnvelope<E>` for error variants). Error-producing methods fold `errorCode` into `message` as `"errorCode: message"` — see [Data Envelope](#data-envelope).

**Success (2xx / 3xx)**

| Method | Status | Description |
|--------|--------|-------------|
| `success(message, data)` | 200 | Create success response |
| `success(message)` | 200 | Success response with no data |
| `ok(message, data)` | 200 | Alias for `success` |
| `created(message, data)` | 201 | Create created response |
| `accepted(message, data)` | 202 | Create accepted response |
| `noContent(message)` | 204 | Create no-content response |
| `notModified(message)` | 304 | Not-modified response (treated as success) |

**Client errors (4xx)**

| Method | Status | Description |
|--------|--------|-------------|
| `badRequest(errorCode, message, data)` | 400 | Create bad request response |
| `unauthorized(errorCode, message, data)` | 401 | Create unauthorized response |
| `paymentRequired(errorCode, message, data)` | 402 | Create payment required response |
| `forbidden(errorCode, message, data)` | 403 | Create forbidden response |
| `notFound(errorCode, message, data)` | 404 | Create not found response |
| `conflict(errorCode, message, data)` | 409 | Create conflict response |
| `gone(errorCode, message, data)` | 410 | Create gone response |
| `unprocessableEntity(errorCode, message, data)` | 422 | Create unprocessable entity response |
| `locked(errorCode, message, data)` | 423 | Create locked response |
| `tooManyRequests(errorCode, message, data)` | 429 | Create too many requests response |

**Server errors (5xx)**

| Method | Status | Description |
|--------|--------|-------------|
| `internalServerError(errorCode, message, data)` | 500 | Create internal server error response |
| `notImplemented(errorCode, message, data)` | 501 | Create not implemented response |
| `badGateway(errorCode, message, data)` | 502 | Create bad gateway response |
| `serviceUnavailable(errorCode, message, data)` | 503 | Create service unavailable response |
| `gatewayTimeout(errorCode, message, data)` | 504 | Create gateway timeout response |

### ApplicationException

```typescript
export abstract class ApplicationError extends Error {
  public readonly errorCode: string;
  public readonly statusCode: number;
  public readonly message: string;
  public readonly cause?: Error;
}
```

## Error Handling Reference

The library provides these error classes:

| Error Class | HTTP Status | Error Code |
|-------------|-------------|------------|
| `ValidationError` | 400 | Custom |
| `MissingFieldsError` | 400 | `MISSING_FIELDS` |
| `InvalidError` | 400 | `INVALID` |
| `AlreadyExistsError` | 409 | `ALREADY_EXISTS` |
| `InsufficientFundsError` | 400 | `INSUFFICIENT_FUNDS` |
| `QuotaExceededError` | 429 | `QUOTA_EXCEEDED` |
| `NotFoundError` | 404 | `NOT_FOUND` |
| `ExpiredError` | 410 | `EXPIRED` |
| `TooManyRequestsError` | 429 | `TOO_MANY_REQUESTS` |
| `PaymentRequiredError` | 402 | `PAYMENT_REQUIRED` |
| `LockedError` | 423 | `LOCKED` |
| `AccountSuspendedError` | 403 | `ACCOUNT_SUSPENDED` |
| `FeatureNotAvailableError` | 501 | `FEATURE_NOT_AVAILABLE` |
| `DataIntegrityError` | 422 | `DATA_INTEGRITY_ERROR` |
| `AuthenticationError` | 401 | Custom |
| `InvalidTokenError` | 401 | `INVALID_TOKEN` |
| `TokenExpiredError` | 401 | `TOKEN_EXPIRED` |
| `SessionExpiredError` | 401 | `SESSION_EXPIRED` |
| `AuthorizationError` | 403 | Custom |
| `PermissionDeniedError` | 403 | `PERMISSION_DENIED` |
| `NotAllowedError` | 403 | `NOT_ALLOWED` |
| `InfrastructureError` | Varies | Custom |
| `ExternalServiceError` | 502 | `EXTERNAL_SERVICE_ERROR` |
| `BadGatewayError` | 502 | `BAD_GATEWAY` |
| `GatewayTimeoutError` | 504 | `GATEWAY_TIMEOUT` |
| `ServiceUnavailableError` | 503 | `SERVICE_UNAVAILABLE` |
| `RequestFailedError` | 500 | `REQUEST_FAILED` |
| `NotImplementedError` | 501 | `NOT_IMPLEMENTED` |
| `MaintenanceModeError` | 503 | `MAINTENANCE_MODE` |

## Compatibility Matrix

| Package Version | Node.js | TypeScript |
|-----------------|---------|------------|
| 1.0.2           | 18+     | 5.0+       |

| Feature | Supported |
|---------|-----------|
| ESM | Yes |
| CommonJS | Via interop |
| Strict mode | Yes |
| Tree-shaking | Yes |

## Contributing

1. Fork the repository
2. Create your feature branch: `git checkout -b feature/my-feature`
3. Commit your changes: `git commit -am 'Add my feature'`
4. Push to the branch: `git push origin feature/my-feature`
5. Submit a pull request

### Development Setup

```bash
# Clone the repository
git clone https://github.com/iamprovy-dev/kernspark-js.git
cd kernspark-js/javascript/packages/kernspark-core

# Install dependencies
npm install

# Build
npm run build

# Run tests
npm test
```

### Code Standards

- Use TypeScript 5.0+
- Follow functional programming patterns where appropriate
- Keep the core framework-independent
- Do not add application-specific logic
- Do not add framework dependencies to core

## License

Apache-2.0

## Author

**Providence Chikukwa**
- Email: iamprovy@outlook.com
- GitHub: https://github.com/iamprovy-dev
- LinkedIn: https://www.linkedin.com/in/provychikukwa
- Organization: [SyncAfrica Business Solutions](https://www.syncafricabs.com)
