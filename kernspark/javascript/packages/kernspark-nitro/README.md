# @syncafricabs/kernspark-nitro

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue)](https://www.typescriptlang.org/)
[![Nitro](https://img.shields.io/badge/Nitro-2.6%2B-00DC82)](https://nitro.unjs.io/)

Nitro adapter for the [SyncAfrica KernSpark](https://github.com/iamprovy-dev/kernspark-js) ecosystem. This package provides production-ready Nitro route handlers, error handlers, and response utilities that integrate the framework-independent `@syncafricabs/kernspark-core` with Nitro.

## Table of Contents

- [What is it?](#what-is-it)
- [Why it exists](#why-it-exists)
- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Comprehensive Usage](#comprehensive-usage)
  - [Route Handlers](#route-handlers)
  - [Error Handling](#error-handling)
  - [Response Adapter](#response-adapter)
  - [With Error Handling Wrapper](#with-error-handling-wrapper)
- [API Reference](#api-reference)
- [Error Handling Reference](#error-handling-reference)
- [TypeScript Support](#typescript-support)
- [Compatibility Matrix](#compatibility-matrix)
- [Contributing](#contributing)
- [License](#license)
- [Author](#author)

## What is it?

`@syncafricabs/kernspark-nitro` is a framework adapter that bridges the [SyncAfrica KernSpark Core](https://github.com/iamprovy-dev/kernspark-js/tree/main/packages/kernspark-core) with Nitro. It provides:

- **NitroIntegration** - Maps `ApplicationError` instances to standardized JSON API responses
- **NitroResponseAdapter** - Helper methods for sending consistent success and error responses
- **withNitroErrorHandling** - Higher-order function for wrapping route handlers with error handling

The adapter follows the **adapter pattern**: it depends on `@syncafricabs/kernspark-core` (the framework-independent layer) and Nitro only in the adapter layer. The core has zero framework dependencies.

## Why it exists

Modern web applications benefit from a **KernSpark** - a common set of contracts, types, and behaviors shared across bounded contexts. The SyncAfrica KernSpark provides:

1. **Standardized API envelopes** (`ApiSuccess` / `ApiError`)
2. **Rich domain error hierarchy** (`ValidationError`, `BusinessError`, `AuthenticationError`, etc.)
3. **Result types** (`Result`, `Ok`, `Err`)
4. **Domain primitives** (`UUID`, `Money`, `Entity`, `ValueObject`)

Without adapters, each team would reimplement framework-specific plumbing to use these core types. This adapter eliminates that duplication by providing Nitro-specific bindings.

## Features

- **Route handler support** - Drop-in Nitro route error handling
- **Standardized JSON responses** - All responses follow the `ApiSuccess` / `ApiError` envelope
- **Zero-configuration error handling** - Wraps route handlers automatically
- **Full TypeScript support** - Comprehensive type definitions included
- **Production-ready** - Proper error handling, stack traces, cause chains
- **Framework isolation** - Core has no Nitro dependencies

## Installation

```bash
npm install @syncafricabs/kernspark-nitro @syncafricabs/kernspark-core nitro
```

Or with your preferred package manager:

```bash
yarn add @syncafricabs/kernspark-nitro @syncafricabs/kernspark-core nitro
```

```bash
pnpm add @syncafricabs/kernspark-nitro @syncafricabs/kernspark-core nitro
```

## Quick Start

```typescript
// routes/users/[id].ts
import { withNitroErrorHandling, NitroResponseAdapter } from '@syncafricabs/kernspark-nitro';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

export default withNitroErrorHandling(async (event) => {
  const id = event.context.params?.id;
  const user = findUser(id);

  if (!user) {
    throw new NotFoundError('User not found');
  }

  return NitroResponseAdapter.ok(event, user, 'User retrieved successfully');
});
```

```typescript
// routes/users.ts
export default withNitroErrorHandling(async (event) => {
  if (event.method === 'POST') {
    const body = await readBody(event);

    if (!body.name || !body.email) {
      throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
    }

    const user = createUser(body);
    return NitroResponseAdapter.created(event, user, 'User created successfully');
  }

  const users = await getUsers();
  return NitroResponseAdapter.ok(event, users, 'Users retrieved successfully');
});
```

## Comprehensive Usage

### Route Handlers

#### GET with Error Handling

```typescript
// routes/users/[id].ts
import { withNitroErrorHandling, NitroResponseAdapter } from '@syncafricabs/kernspark-nitro';
import { NotFoundError } from '@syncafricabs/kernspark-core';

export default withNitroErrorHandling(async (event) => {
  const id = event.context.params?.id;
  const user = findUser(id);

  if (!user) {
    throw new NotFoundError(`User with id ${id} not found`);
  }

  return NitroResponseAdapter.ok(event, user, 'User retrieved successfully');
});
```

#### POST with Validation

```typescript
// routes/users.ts
import { withNitroErrorHandling, NitroResponseAdapter } from '@syncafricabs/kernspark-nitro';
import { ValidationError, ConflictError, BusinessError } from '@syncafricabs/kernspark-core';

export default withNitroErrorHandling(async (event) => {
  const body = await readBody(event);

  if (!body.name || !body.email) {
    throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
  }

  if (!isValidEmail(body.email)) {
    throw new ValidationError('INVALID_EMAIL', 'Email format is invalid');
  }

  const existingUser = findUserByEmail(body.email);
  if (existingUser) {
    throw new ConflictError('User with this email already exists');
  }

  if (body.balance < 0) {
    throw new BusinessError('INVALID_BALANCE', 400, 'Balance cannot be negative');
  }

  const user = createUser(body);
  return NitroResponseAdapter.created(event, user, 'User created successfully');
});
```

### Error Handling

#### Custom Error Handler Configuration

```typescript
import { createNitroErrorHandler, NitroErrorHandlerOptions } from '@syncafricabs/kernspark-nitro';

const options: NitroErrorHandlerOptions = {
  logErrors: true,
  logStack: true,
  includeCause: true,
  includeStack: process.env.NODE_ENV === 'development',
  defaultStatusCode: 500,
  logger: (error, context) => {
    console.error({
      method: context.method,
      path: context.path,
      params: context.params,
      error: error.message,
      stack: error.stack,
    });
  },
};

export default withNitroErrorHandling(handler, options);
```

### Response Adapter

#### Success Responses

```typescript
import { NitroResponseAdapter } from '@syncafricabs/kernspark-nitro';

// 200 OK with data
NitroResponseAdapter.ok(event, user, 'User retrieved');

// 201 Created with data
NitroResponseAdapter.created(event, user, 'User created');

// 204 No Content
NitroResponseAdapter.noContent(event);
```

#### Error Responses

```typescript
import { NitroResponseAdapter } from '@syncafricabs/kernspark-nitro';

NitroResponseAdapter.badRequest(event, 'MISSING_FIELDS', 'Email and password are required');
NitroResponseAdapter.unauthorized(event, 'INVALID_CREDENTIALS', 'Invalid email or password');
NitroResponseAdapter.forbidden(event, 'PERMISSION_DENIED', 'You do not have permission');
NitroResponseAdapter.notFound(event, 'NOT_FOUND', 'User not found');
NitroResponseAdapter.conflict(event, 'CONFLICT', 'Resource already exists');
NitroResponseAdapter.unprocessableEntity(event, 'INVALID_DATA', 'Invalid request data');
NitroResponseAdapter.tooManyRequests(event, 'RATE_LIMIT_EXCEEDED', 'Too many requests');
NitroResponseAdapter.internalServerError(event, 'INTERNAL_ERROR', 'An unexpected error occurred');
NitroResponseAdapter.badGateway(event, 'BAD_GATEWAY', 'Upstream service error');
NitroResponseAdapter.serviceUnavailable(event, 'SERVICE_UNAVAILABLE', 'Service temporarily unavailable');
NitroResponseAdapter.gatewayTimeout(event, 'GATEWAY_TIMEOUT', 'Upstream service timeout');
```

### With Error Handling Wrapper

The `withNitroErrorHandling` function wraps route handlers and automatically catches `ApplicationError` instances:

```typescript
import { withNitroErrorHandling, NitroResponseAdapter } from '@syncafricabs/kernspark-nitro';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

export default withNitroErrorHandling(async (event) => {
  const id = event.context.params?.id;
  const user = findUser(id);

  if (!user) {
    throw new NotFoundError('User not found');
  }

  return NitroResponseAdapter.ok(event, user);
});
```

## API Reference

### `nitroErrorHandler`

Nitro error handling handler. Catches all errors and maps them to standardized JSON responses.

```typescript
function nitroErrorHandler(error: Error, event: any): void;
```

### `createNitroErrorHandler(options?: NitroErrorHandlerOptions)`

Creates a customized error handler.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `logErrors` | `boolean` | `true` | Enable/disable error logging |
| `logStack` | `boolean` | `false` | Include stack traces in logs |
| `includeCause` | `boolean` | `true` | Include error cause in response |
| `includeStack` | `boolean` | `false` | Include stack traces in response (dev only) |
| `defaultStatusCode` | `number` | `500` | Default status for unknown errors |
| `logger` | `function` | `console.error` | Custom logger function |

### `NitroResponseAdapter`

Static utility class for sending responses.

| Method | HTTP Status | Description |
|--------|-------------|-------------|
| `ok(event, data?, message?)` | 200 | Send success response with data |
| `created(event, data?, message?)` | 201 | Send created response with data |
| `noContent(event, message?)` | 204 | Send no content response |
| `sendSuccess(event, apiSuccess)` | varies | Send raw ApiSuccess |
| `sendError(event, apiError)` | varies | Send raw ApiError |
| `badRequest(event, errorCode, message)` | 400 | Send bad request error |
| `unauthorized(event, errorCode, message)` | 401 | Send unauthorized error |
| `forbidden(event, errorCode, message)` | 403 | Send forbidden error |
| `notFound(event, errorCode, message)` | 404 | Send not found error |
| `conflict(event, errorCode, message)` | 409 | Send conflict error |
| `unprocessableEntity(event, errorCode, message)` | 422 | Send unprocessable entity error |
| `tooManyRequests(event, errorCode, message)` | 429 | Send too many requests error |
| `internalServerError(event, errorCode, message)` | 500 | Send internal server error |
| `badGateway(event, errorCode, message)` | 502 | Send bad gateway error |
| `serviceUnavailable(event, errorCode, message)` | 503 | Send service unavailable error |
| `gatewayTimeout(event, errorCode, message)` | 504 | Send gateway timeout error |

### `withNitroErrorHandling(handler, options?)`

Wraps a Nitro route handler with error handling.

```typescript
function withNitroErrorHandling<T extends NitroRouteHandler>(
  handler: T,
  options?: NitroErrorHandlerOptions
): NitroRouteHandler;
```

## Error Handling Reference

The error handler maps the following `ApplicationError` subclasses:

| Error Class | HTTP Status | Error Code |
|-------------|-------------|------------|
| `ValidationError` | 400 | Custom (e.g., `INVALID`, `MISSING_FIELDS`) |
| `MissingFieldsError` | 400 | `MISSING_FIELDS` |
| `BusinessError` | Varies | Custom |
| `ConflictError` | 409 | `CONFLICT` |
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
| `AuthenticationError` | 401 | Custom (e.g., `INVALID_TOKEN`) |
| `InvalidTokenError` | 401 | `INVALID_TOKEN` |
| `TokenExpiredError` | 401 | `TOKEN_EXPIRED` |
| `SessionExpiredError` | 401 | `SESSION_EXPIRED` |
| `AuthorizationError` | Varies | Custom |
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

Unknown `ApplicationError` instances use their embedded `statusCode`. Non-`ApplicationError` errors return HTTP 500 with `INTERNAL_SERVER_ERROR` code.

## TypeScript Support

Full TypeScript support with strict mode enabled. The adapter extends Nitro types:

```typescript
// Nitro event type augmentation
interface NitroRouteContext {
  method: string;
  path: string;
  context: any;
}
```

### Strict Mode

This package is compiled with `strict: true`. Enable strict mode in your project for full type safety:

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true
  }
}
```

## Compatibility Matrix

| Package Version | Node.js | Nitro | @syncafricabs/kernspark-core |
|----------------|---------|-------|----------------------------------|
| 1.0.0 | >=18.0.0 | ^2.6.0 | ^1.0.0 |

| Feature | Nitro 2.6+ |
|---------|-------------|
| Route handlers | Supported |
| Error handling | Supported |
| Response utilities | Supported |
| TypeScript 5.0+ | Supported |

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
cd kernspark/packages/kernspark-nitro

# Install dependencies
npm install

# Build
npm run build

# Run lint
npm run lint

# Run tests
npm test
```

### Code Standards

- Use TypeScript with strict mode
- Follow the existing code style (2-space indentation, semicolons)
- Ensure all tests pass before submitting PR

## License

Apache-2.0

## Author

**Providence Chikukwa**
- Email: iamprovy@outlook.com
- GitHub: https://github.com/iamprovy-dev
- LinkedIn: https://www.linkedin.com/in/provychikukwa
- Organization: [SyncAfrica Business Solutions](https://www.syncafricabs.com)


