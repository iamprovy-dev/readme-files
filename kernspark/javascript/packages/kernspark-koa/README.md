# @syncafricabs/kernspark-koa

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue)](https://www.typescriptlang.org/)
[![Koa](https://img.shields.io/badge/Koa-2.x-green)](https://koajs.com/)

Koa adapter for the [SyncAfrica KernSpark](https://github.com/iamprovy-dev/kernspark-js) ecosystem. This package provides production-ready Koa middleware, error handlers, and response utilities that integrate the framework-independent `@syncafricabs/kernspark-core` with Koa.

## Table of Contents

- [What is it?](#what-is-it)
- [Why it exists](#why-it-exists)
- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Comprehensive Usage](#comprehensive-usage)
  - [Middleware](#middleware)
  - [Error Handling](#error-handling)
  - [Response Adapter](#response-adapter)
  - [Complete Server Example](#complete-server-example)
- [API Reference](#api-reference)
- [Error Handling Reference](#error-handling-reference)
- [TypeScript Support](#typescript-support)
- [Compatibility Matrix](#compatibility-matrix)
- [Contributing](#contributing)
- [License](#license)
- [Author](#author)

## What is it?

`@syncafricabs/kernspark-koa` is a framework adapter that bridges the [SyncAfrica KernSpark Core](https://github.com/iamprovy-dev/kernspark-js/tree/main/packages/kernspark-core) with Koa. It provides:

- **KoaIntegration** - Maps `ApplicationError` instances to standardized JSON API responses
- **KoaResponseAdapter** - Helper methods for sending consistent success and error responses
- **koaMiddleware** - Request/response logging and correlation ID tracking
- **Error handling** - Centralized error handling for Koa applications

The adapter follows the **adapter pattern**: it depends on `@syncafricabs/kernspark-core` (the framework-independent layer) and Koa only in the adapter layer. The core has zero framework dependencies.

## Why it exists

Modern web applications benefit from a **KernSpark** - a common set of contracts, types, and behaviors shared across bounded contexts. The SyncAfrica KernSpark provides:

1. **Standardized API envelopes** (`ApiSuccess` / `ApiError`)
2. **Rich domain error hierarchy** (`ValidationError`, `BusinessError`, `AuthenticationError`, etc.)
3. **Result types** (`Result`, `Ok`, `Err`)
4. **Domain primitives** (`UUID`, `Money`, `Entity`, `ValueObject`)

Without adapters, each team would reimplement framework-specific plumbing to use these core types. This adapter eliminates that duplication by providing Koa-specific bindings.

## Features

- **Middleware support** - Drop-in Koa middleware for correlation ID and logging
- **Standardized JSON responses** - All responses follow the `ApiSuccess` / `ApiError` envelope
- **Zero-configuration error handling** - Centralized error handler for Koa
- **Full TypeScript support** - Comprehensive type definitions included
- **Production-ready** - Proper error handling, stack traces, cause chains
- **Framework isolation** - Core has no Koa dependencies

## Installation

```bash
npm install @syncafricabs/kernspark-koa @syncafricabs/kernspark-core koa
```

Or with your preferred package manager:

```bash
yarn add @syncafricabs/kernspark-koa @syncafricabs/kernspark-core koa
```

```bash
pnpm add @syncafricabs/kernspark-koa @syncafricabs/kernspark-core koa
```

## Quick Start

```typescript
import Koa from 'koa';
import { koaMiddleware, koaErrorHandler, KoaResponseAdapter } from '@syncafricabs/kernspark-koa';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

const app = new Koa();

app.use(koaMiddleware({
  correlationIdHeader: 'x-correlation-id',
  logRequests: true,
  logResponses: true,
}));

app.use(async (ctx, next) => {
  try {
    await next();
  } catch (error) {
    koaErrorHandler(error, ctx, () => {});
  }
});

app.get('/users/:id', async (ctx) => {
  const user = findUser(ctx.params.id);

  if (!user) {
    throw new NotFoundError('User not found');
  }

  KoaResponseAdapter.ok(ctx, user, 'User retrieved successfully');
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

## Comprehensive Usage

### Middleware

#### Correlation ID Middleware

The correlation ID middleware generates or propagates a unique identifier for each request.

```typescript
import { koaMiddleware, KoaMiddlewareOptions } from '@syncafricabs/kernspark-koa';

const options: KoaMiddlewareOptions = {
  correlationIdHeader: 'x-correlation-id',
  logRequests: true,
  logResponses: true,
  logErrors: true,
  sensitiveFields: ['password', 'token', 'secret', 'apiKey', 'creditCard'],
  logger: (meta) => {
    console.log(JSON.stringify(meta));
  },
};

app.use(koaMiddleware(options));
```

Sample log output:

```json
{
  "method": "GET",
  "url": "/api/users",
  "correlationId": "550e8400-e29b-41d4-a716-446655440000"
}
```

```json
{
  "method": "POST",
  "url": "/api/users",
  "statusCode": 201,
  "contentLength": 156,
  "responseTime": 45,
  "correlationId": "550e8400-e29b-41d4-a716-446655440000"
}
```

### Error Handling

#### Basic Setup

```typescript
import { koaErrorHandler } from '@syncafricabs/kernspark-koa';

app.use(async (ctx, next) => {
  try {
    await next();
  } catch (error) {
    koaErrorHandler(error, ctx, () => {});
  }
});
```

#### Custom Error Handler Configuration

```typescript
import { createKoaErrorHandler, KoaErrorHandlerOptions } from '@syncafricabs/kernspark-koa';

const options: KoaErrorHandlerOptions = {
  logErrors: true,
  logStack: true,
  includeCause: true,
  includeStack: process.env.NODE_ENV === 'development',
  defaultStatusCode: 500,
  logger: (error, ctx) => {
    console.error({
      correlationId: ctx.correlationId,
      method: ctx.method,
      url: ctx.url,
      error: error.message,
      stack: error.stack,
    });
  },
};

app.use(async (ctx, next) => {
  try {
    await next();
  } catch (error) {
    createKoaErrorHandler(options)(error, ctx, () => {});
  }
});
```

#### Throwing Errors from Route Handlers

```typescript
import {
  ValidationError,
  MissingFieldsError,
  BusinessError,
  ConflictError,
  NotFoundError,
  AuthenticationError,
  InvalidTokenError,
  AuthorizationError,
  PermissionDeniedError,
  InfrastructureError,
  ExternalServiceError,
} from '@syncafricabs/kernspark-core';

app.post('/users', async (ctx) => {
  const { name, email } = ctx.request.body;

  if (!name || !email) {
    throw new MissingFieldsError('Name and email are required');
  }

  if (!isValidEmail(email)) {
    throw new ValidationError('INVALID_EMAIL', 'Email format is invalid');
  }

  if (userExists(email)) {
    throw new ConflictError('User with this email already exists');
  }

  const user = createUser(ctx.request.body);
  KoaResponseAdapter.created(ctx, user, 'User created successfully');
});

app.get('/users/:id', async (ctx) => {
  const user = findUser(ctx.params.id);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  KoaResponseAdapter.ok(ctx, user);
});

app.delete('/users/:id', async (ctx) => {
  if (!ctx.state.user) {
    throw new InvalidTokenError('Invalid or expired token');
  }

  if (!ctx.state.user.canDeleteUsers) {
    throw new PermissionDeniedError('You do not have permission to delete users');
  }

  deleteUser(ctx.params.id);
  KoaResponseAdapter.noContent(ctx, 'User deleted successfully');
});
```

### Response Adapter

#### Success Responses

```typescript
import { KoaResponseAdapter } from '@syncafricabs/kernspark-koa';

// 200 OK with data
KoaResponseAdapter.ok(ctx, user, 'User retrieved');

// 201 Created with data
KoaResponseAdapter.created(ctx, user, 'User created');

// 204 No Content
KoaResponseAdapter.noContent(ctx);

// Send raw ApiSuccess
const success = { status: 200, success: true, data: users, message: 'OK' };
KoaResponseAdapter.sendSuccess(ctx, success);
```

#### Error Responses

```typescript
import { KoaResponseAdapter } from '@syncafricabs/kernspark-koa';

KoaResponseAdapter.badRequest(ctx, 'MISSING_FIELDS', 'Email and password are required');
KoaResponseAdapter.unauthorized(ctx, 'INVALID_CREDENTIALS', 'Invalid email or password');
KoaResponseAdapter.forbidden(ctx, 'PERMISSION_DENIED', 'You do not have permission');
KoaResponseAdapter.notFound(ctx, 'NOT_FOUND', 'User not found');
KoaResponseAdapter.conflict(ctx, 'CONFLICT', 'Resource already exists');
KoaResponseAdapter.unprocessableEntity(ctx, 'INVALID_DATA', 'Invalid request data');
KoaResponseAdapter.tooManyRequests(ctx, 'RATE_LIMIT_EXCEEDED', 'Too many requests');
KoaResponseAdapter.internalServerError(ctx, 'INTERNAL_ERROR', 'An unexpected error occurred');
KoaResponseAdapter.badGateway(ctx, 'BAD_GATEWAY', 'Upstream service error');
KoaResponseAdapter.serviceUnavailable(ctx, 'SERVICE_UNAVAILABLE', 'Service temporarily unavailable');
KoaResponseAdapter.gatewayTimeout(ctx, 'GATEWAY_TIMEOUT', 'Upstream service timeout');
```

### Complete Server Example

```typescript
import Koa from 'koa';
import { koaMiddleware, koaErrorHandler, KoaResponseAdapter } from '@syncafricabs/kernspark-koa';
import { NotFoundError, ValidationError, ConflictError } from '@syncafricabs/kernspark-core';

const app = new Koa();

// Middleware
app.use(koaMiddleware({
  correlationIdHeader: 'x-correlation-id',
  logRequests: true,
  logResponses: true,
  sensitiveFields: ['password', 'token', 'secret'],
}));

// Error handling
app.use(async (ctx, next) => {
  try {
    await next();
  } catch (error) {
    koaErrorHandler(error, ctx, () => {});
  }
});

// Routes
app.get('/users/:id', async (ctx) => {
  const user = findUser(ctx.params.id);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  KoaResponseAdapter.ok(ctx, user, 'User retrieved successfully');
});

app.post('/users', async (ctx) => {
  const { name, email } = ctx.request.body;

  if (!name || !email) {
    throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
  }

  if (findUserByEmail(email)) {
    throw new ConflictError('User with this email already exists');
  }

  const user = createUser(ctx.request.body);
  KoaResponseAdapter.created(ctx, user, 'User created successfully');
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

## API Reference

### `koaErrorHandler`

Koa error handling middleware. Catches all errors and maps them to standardized JSON responses.

```typescript
function koaErrorHandler(error: Error, ctx: Context, next: Next): void;
```

### `createKoaErrorHandler(options?: KoaErrorHandlerOptions)`

Creates a customized error handler.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `logErrors` | `boolean` | `true` | Enable/disable error logging |
| `logStack` | `boolean` | `false` | Include stack traces in logs |
| `includeCause` | `boolean` | `true` | Include error cause in response |
| `includeStack` | `boolean` | `false` | Include stack traces in response (dev only) |
| `defaultStatusCode` | `number` | `500` | Default status for unknown errors |
| `logger` | `function` | `console.error` | Custom logger function |

### `KoaResponseAdapter`

Static utility class for sending responses.

| Method | HTTP Status | Description |
|--------|-------------|-------------|
| `ok(ctx, data?, message?)` | 200 | Send success response with data |
| `created(ctx, data?, message?)` | 201 | Send created response with data |
| `noContent(ctx, message?)` | 204 | Send no content response |
| `sendSuccess(ctx, apiSuccess)` | varies | Send raw ApiSuccess |
| `sendError(ctx, apiError)` | varies | Send raw ApiError |
| `badRequest(ctx, errorCode, message)` | 400 | Send bad request error |
| `unauthorized(ctx, errorCode, message)` | 401 | Send unauthorized error |
| `forbidden(ctx, errorCode, message)` | 403 | Send forbidden error |
| `notFound(ctx, errorCode, message)` | 404 | Send not found error |
| `conflict(ctx, errorCode, message)` | 409 | Send conflict error |
| `unprocessableEntity(ctx, errorCode, message)` | 422 | Send unprocessable entity error |
| `tooManyRequests(ctx, errorCode, message)` | 429 | Send too many requests error |
| `internalServerError(ctx, errorCode, message)` | 500 | Send internal server error |
| `badGateway(ctx, errorCode, message)` | 502 | Send bad gateway error |
| `serviceUnavailable(ctx, errorCode, message)` | 503 | Send service unavailable error |
| `gatewayTimeout(ctx, errorCode, message)` | 504 | Send gateway timeout error |

### `koaMiddleware(options?: KoaMiddlewareOptions)`

Koa middleware for correlation ID and request/response logging.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `correlationIdHeader` | `string` | `'x-correlation-id'` | Header name for correlation ID |
| `logRequests` | `boolean` | `true` | Log incoming requests |
| `logResponses` | `boolean` | `true` | Log outgoing responses |
| `logErrors` | `boolean` | `true` | Log errors |
| `sensitiveFields` | `string[]` | `['password', 'token', 'secret', ...]` | Body fields to redact |
| `logger` | `function` | `console.info` | Custom logger function |

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

Full TypeScript support with strict mode enabled. The adapter extends Koa types:

```typescript
import { Context, Next } from 'koa';

// Koa context augmentation for correlation ID
declare module 'koa' {
  interface Context {
    correlationId?: string;
  }
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

| Package Version | Node.js | Koa | @syncafricabs/kernspark-core |
|----------------|---------|-----|----------------------------------|
| 1.0.0 | >=18.0.0 | ^2.14.0 | ^1.0.0 |

| Feature | Koa 2.14+ |
|---------|-------------|
| Middleware | Supported |
| Error handling | Supported |
| Request/response logging | Supported |
| Correlation ID propagation | Supported |
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
cd kernspark/packages/kernspark-koa

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


