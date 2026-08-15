# @syncafricabs/kernspark-elysiajs

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue)](https://www.typescriptlang.org/)
[![Elysia](https://img.shields.io/badge/Elysia-0.7%2B-8C38E9)](https://elysiajs.com/)

ElysiaJS adapter for the [SyncAfrica KernSpark](https://github.com/iamprovy-dev/kernspark-js) ecosystem. This package provides production-ready ElysiaJS plugins, error handlers, and response utilities that integrate the framework-independent `@syncafricabs/kernspark-core` with ElysiaJS.

## Table of Contents

- [What is it?](#what-is-it)
- [Why it exists](#why-it-exists)
- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Comprehensive Usage](#comprehensive-usage)
  - [Plugin Registration](#plugin-registration)
  - [Error Handling](#error-handling)
  - [Response Adapter](#response-adapter)
  - [Route Handlers](#route-handlers)
- [API Reference](#api-reference)
- [Error Handling Reference](#error-handling-reference)
- [TypeScript Support](#typescript-support)
- [Compatibility Matrix](#compatibility-matrix)
- [Contributing](#contributing)
- [License](#license)
- [Author](#author)

## What is it?

`@syncafricabs/kernspark-elysiajs` is a framework adapter that bridges the [SyncAfrica KernSpark Core](https://github.com/iamprovy-dev/kernspark-js/tree/main/packages/kernspark-core) with ElysiaJS. It provides:

- **ElysiaJsIntegration** - Maps `ApplicationError` instances to standardized JSON API responses
- **ElysiaJsResponseAdapter** - Helper methods for sending consistent success and error responses
- **elysiaJsPlugin** - Elysia plugin with correlation ID and error handling
- **withElysiaErrorHandling** - Higher-order function for wrapping route handlers with error handling

The adapter follows the **adapter pattern**: it depends on `@syncafricabs/kernspark-core` (the framework-independent layer) and ElysiaJS only in the adapter layer. The core has zero framework dependencies.

## Why it exists

Modern web applications benefit from a **KernSpark** - a common set of contracts, types, and behaviors shared across bounded contexts. The SyncAfrica KernSpark provides:

1. **Standardized API envelopes** (`ApiSuccess` / `ApiError`)
2. **Rich domain error hierarchy** (`ValidationError`, `BusinessError`, `AuthenticationError`, etc.)
3. **Result types** (`Result`, `Ok`, `Err`)
4. **Domain primitives** (`UUID`, `Money`, `Entity`, `ValueObject`)

Without adapters, each team would reimplement framework-specific plumbing to use these core types. This adapter eliminates that duplication by providing ElysiaJS-specific bindings.

## Features

- **Plugin registration** - Drop-in Elysia plugin with error handling
- **Standardized JSON responses** - All responses follow the `ApiSuccess` / `ApiError` envelope
- **Zero-configuration error handling** - Wraps route handlers automatically
- **Full TypeScript support** - Comprehensive type definitions included
- **Production-ready** - Proper error handling, stack traces, cause chains
- **Framework isolation** - Core has no ElysiaJS dependencies

## Installation

```bash
npm install @syncafricabs/kernspark-elysiajs @syncafricabs/kernspark-core elysia
```

Or with your preferred package manager:

```bash
yarn add @syncafricabs/kernspark-elysiajs @syncafricabs/kernspark-core elysia
```

```bash
pnpm add @syncafricabs/kernspark-elysiajs @syncafricabs/kernspark-core elysia
```

## Quick Start

```typescript
import { Elysia } from 'elysia';
import { elysiaJsPlugin, withElysiaErrorHandling, ElysiaJsResponseAdapter } from '@syncafricabs/kernspark-elysiajs';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

const app = new Elysia()
  .use(elysiaJsPlugin({
    logErrors: true,
    includeCause: true,
  }))
  .get('/users/:id', withElysiaErrorHandling(async ({ params }) => {
    const user = findUser(params.id);

    if (!user) {
      throw new NotFoundError('User not found');
    }

    return ElysiaJsResponseAdapter.ok({ status: 200, success: true, data: user, message: 'OK' });
  }))
  .post('/users', withElysiaErrorHandling(async ({ body }) => {
    if (!body.name || !body.email) {
      throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
    }

    const user = createUser(body);
    return ElysiaJsResponseAdapter.created({ status: 201, success: true, data: user, message: 'Created' });
  }))
  .listen(3000);
```

## Comprehensive Usage

### Plugin Registration

#### Basic Plugin

```typescript
import { Elysia } from 'elysia';
import { elysiaJsPlugin } from '@syncafricabs/kernspark-elysiajs';

const app = new Elysia()
  .use(elysiaJsPlugin())
  .get('/', () => 'Hello World')
  .listen(3000);
```

#### Plugin with Options

```typescript
import { Elysia } from 'elysia';
import { elysiaJsPlugin, ElysiaJsPluginOptions } from '@syncafricabs/kernspark-elysiajs';

const options: ElysiaJsPluginOptions = {
  logErrors: true,
  logStack: process.env.NODE_ENV === 'development',
  includeCause: true,
  includeStack: false,
  defaultStatusCode: 500,
  logger: (error, context) => {
    console.error({
      method: context.method,
      path: context.path,
      error: error.message,
      stack: error.stack,
    });
  },
};

const app = new Elysia()
  .use(elysiaJsPlugin(options))
  .get('/', () => 'Hello World')
  .listen(3000);
```

### Error Handling

#### Throwing Errors from Route Handlers

```typescript
import { withElysiaErrorHandling, ElysiaJsResponseAdapter } from '@syncafricabs/kernspark-elysiajs';
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

const app = new Elysia()
  .get('/users/:id', withElysiaErrorHandling(async ({ params }) => {
    const user = findUser(params.id);
    if (!user) {
      throw new NotFoundError('User not found');
    }
    return { user, message: 'User retrieved' };
  }))
  .post('/users', withElysiaErrorHandling(async ({ body }) => {
    if (!body.name || !body.email) {
      throw new MissingFieldsError('Name and email are required');
    }
    const user = createUser(body);
    return { user, message: 'User created' };
  }))
  .delete('/users/:id', withElysiaErrorHandling(async ({ params, headers }) => {
    const token = headers.authorization;
    if (!token || !isValidToken(token)) {
      throw new InvalidTokenError('Invalid or expired token');
    }
    deleteUser(params.id);
    return { message: 'User deleted' };
  }));
```

### Response Adapter

#### Success Responses

```typescript
import { ElysiaJsResponseAdapter } from '@syncafricabs/kernspark-elysiajs';

// 200 OK with data
return ElysiaJsResponseAdapter.ok(user, 'User retrieved');

// 201 Created with data
return ElysiaJsResponseAdapter.created(user, 'User created');

// 204 No Content
return ElysiaJsResponseAdapter.noContent();
```

#### Error Responses

```typescript
import { ElysiaJsResponseAdapter } from '@syncafricabs/kernspark-elysiajs';

return ElysiaJsResponseAdapter.badRequest('MISSING_FIELDS', 'Email and password are required');
return ElysiaJsResponseAdapter.unauthorized('INVALID_CREDENTIALS', 'Invalid email or password');
return ElysiaJsResponseAdapter.forbidden('PERMISSION_DENIED', 'You do not have permission');
return ElysiaJsResponseAdapter.notFound('NOT_FOUND', 'User not found');
return ElysiaJsResponseAdapter.conflict('CONFLICT', 'Resource already exists');
return ElysiaJsResponseAdapter.unprocessableEntity('INVALID_DATA', 'Invalid request data');
return ElysiaJsResponseAdapter.tooManyRequests('RATE_LIMIT_EXCEEDED', 'Too many requests');
return ElysiaJsResponseAdapter.internalServerError('INTERNAL_ERROR', 'An unexpected error occurred');
return ElysiaJsResponseAdapter.badGateway('BAD_GATEWAY', 'Upstream service error');
return ElysiaJsResponseAdapter.serviceUnavailable('SERVICE_UNAVAILABLE', 'Service temporarily unavailable');
return ElysiaJsResponseAdapter.gatewayTimeout('GATEWAY_TIMEOUT', 'Upstream service timeout');
```

### Route Handlers

#### GET with Validation

```typescript
import { withElysiaErrorHandling, ElysiaJsResponseAdapter } from '@syncafricabs/kernspark-elysiajs';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

const app = new Elysia()
  .get('/users/:id', withElysiaErrorHandling(async ({ params }) => {
    const user = findUser(params.id);

    if (!user) {
      throw new NotFoundError('User not found');
    }

    return ElysiaJsResponseAdapter.ok({ status: 200, success: true, data: user, message: 'OK' });
  }));
```

#### POST with Business Rules

```typescript
import { withElysiaErrorHandling, ElysiaJsResponseAdapter } from '@syncafricabs/kernspark-elysiajs';
import { ValidationError, ConflictError, BusinessError } from '@syncafricabs/kernspark-core';

const app = new Elysia()
  .post('/users', withElysiaErrorHandling(async ({ body }) => {
    if (!body.name || !body.email) {
      throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
    }

    const existingUser = findUserByEmail(body.email);
    if (existingUser) {
      throw new ConflictError('User with this email already exists');
    }

    if (body.balance < 0) {
      throw new BusinessError('INVALID_BALANCE', 400, 'Balance cannot be negative');
    }

    const user = createUser(body);
    return ElysiaJsResponseAdapter.created({ status: 201, success: true, data: user, message: 'Created' });
  }));
```

## API Reference

### `elysiaJsErrorHandler`

ElysiaJS error handling handler. Catches all errors and maps them to standardized JSON responses.

```typescript
function elysiaJsErrorHandler(error: Error, context: { method: string; path: string }): Response;
```

### `createElysiaJsErrorHandler(options?: ElysiaJsErrorHandlerOptions)`

Creates a customized error handler.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `logErrors` | `boolean` | `true` | Enable/disable error logging |
| `logStack` | `boolean` | `false` | Include stack traces in logs |
| `includeCause` | `boolean` | `true` | Include error cause in response |
| `includeStack` | `boolean` | `false` | Include stack traces in response (dev only) |
| `defaultStatusCode` | `number` | `500` | Default status for unknown errors |
| `logger` | `function` | `console.error` | Custom logger function |

### `ElysiaJsResponseAdapter`

Static utility class for sending responses.

| Method | HTTP Status | Description |
|--------|-------------|-------------|
| `sendSuccess(response, apiSuccess)` | varies | Send raw ApiSuccess |
| `sendError(response, apiError)` | varies | Send raw ApiError |
| `ok(data?, message?)` | 200 | Send success response with data |
| `created(data?, message?)` | 201 | Send created response with data |
| `noContent(message?)` | 204 | Send no content response |
| `badRequest(errorCode, message)` | 400 | Send bad request error |
| `unauthorized(errorCode, message)` | 401 | Send unauthorized error |
| `forbidden(errorCode, message)` | 403 | Send forbidden error |
| `notFound(errorCode, message)` | 404 | Send not found error |
| `conflict(errorCode, message)` | 409 | Send conflict error |
| `unprocessableEntity(errorCode, message)` | 422 | Send unprocessable entity error |
| `tooManyRequests(errorCode, message)` | 429 | Send too many requests error |
| `internalServerError(errorCode, message)` | 500 | Send internal server error |
| `badGateway(errorCode, message)` | 502 | Send bad gateway error |
| `serviceUnavailable(errorCode, message)` | 503 | Send service unavailable error |
| `gatewayTimeout(errorCode, message)` | 504 | Send gateway timeout error |

### `elysiaJsPlugin(options?: ElysiaJsPluginOptions)`

Creates an Elysia plugin with error handling and correlation ID support.

```typescript
function elysiaJsPlugin(options?: ElysiaJsPluginOptions): Elysia;
```

### `withElysiaErrorHandling(handler, options?)`

Wraps an Elysia route handler with error handling.

```typescript
function withElysiaErrorHandling<T extends ElysiaJsRouteHandler>(
  handler: T,
  options?: ElysiaJsErrorHandlerOptions
): ElysiaJsRouteHandler;
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

Full TypeScript support with strict mode enabled. The adapter extends ElysiaJS types:

```typescript
import { Elysia } from 'elysia';

// Route handler type augmentation
interface RouteHandlerContext {
  body: any;
  params: Record<string, string>;
  headers: Headers;
  request: Request;
  set: {
    headers: Headers;
    status: number;
  };
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

| Package Version | Node.js | ElysiaJS | @syncafricabs/kernspark-core |
|----------------|---------|----------|----------------------------------|
| 1.0.0 | >=18.0.0 | ^0.7.0 | ^1.0.0 |

| Feature | ElysiaJS 0.7+ |
|---------|---------------|
| Plugin system | Supported |
| Error handling | Supported |
| Route handlers | Supported |
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
cd kernspark/packages/kernspark-elysiajs

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


