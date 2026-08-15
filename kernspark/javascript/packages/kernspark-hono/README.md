# @syncafricabs/kernspark-hono

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue)](https://www.typescriptlang.org/)
[![Hono](https://img.shields.io/badge/Hono-3.x-green)](https://hono.dev/)

Hono adapter for the [SyncAfrica KernSpark](https://github.com/iamprovy-dev/kernspark-js) ecosystem. This package provides production-ready Hono middleware, error handlers, and response utilities that integrate the framework-independent `@syncafricabs/kernspark-core` with Hono.

## Table of Contents

- [What is it?](#what-is-it)
- [Why it exists](#why-it-exists)
- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Comprehensive Usage](#comprehensive-usage)
  - [Error Handling](#error-handling)
  - [Response Adapter](#response-adapter)
  - [Middleware](#middleware)
- [API Reference](#api-reference)
- [Error Handling Reference](#error-handling-reference)
- [TypeScript Support](#typescript-support)
- [Compatibility Matrix](#compatibility-matrix)
- [Contributing](#contributing)
- [License](#license)

## What is it?

`@syncafricabs/kernspark-hono` is a framework adapter that bridges the [SyncAfrica KernSpark Core](https://github.com/iamprovy-dev/kernspark-js/tree/main/packages/kernspark-core) with Hono. It provides:

- **errors** - Hono error handler middleware that maps `ApplicationError` instances to standardized JSON API responses
- **response** - Helper methods for sending consistent success and error responses
- **middleware** - Request/response logging and correlation ID tracking middleware

The adapter follows the **adapter pattern**: it depends on `@syncafricabs/kernspark-core` (the framework-independent layer) and Hono only in the adapter layer. The core has zero framework dependencies.

## Why it exists

Modern microservice architectures benefit from a **KernSpark** - a common set of contracts, types, and behaviors shared across bounded contexts. The SyncAfrica KernSpark provides:

1. **Standardized API envelopes** (`ApiSuccess` / `ApiError`)
2. **Rich domain error hierarchy** (`ValidationError`, `BusinessError`, `AuthenticationError`, etc.)
3. **Result types** (`Result`, `Ok`, `Err`)
4. **Domain primitives** (`UUID`, `Money`, `Entity`, `ValueObject`)

Without adapters, each team would reimplement framework-specific plumbing to use these core types. This adapter eliminates that duplication by providing Hono-specific bindings.

## Features

- **Error handling middleware** - Catch and format errors consistently
- **Standardized JSON responses** - All responses follow the `ApiSuccess` / `ApiError` envelope
- **Correlation ID propagation** - Automatic `x-correlation-id` header management
- **Request/response logging** - Configurable structured logging with sanitization
- **Full TypeScript support** - Comprehensive type definitions included
- **Edge-ready** - Works in Cloudflare Workers, Deno, Bun, Node.js
- **Production-ready** - Proper error handling, stack traces, cause chains
- **Framework isolation** - Core has no Hono dependencies

## Installation

```bash
npm install @syncafricabs/kernspark-hono @syncafricabs/kernspark-core hono
```

Or with your preferred package manager:

```bash
yarn add @syncafricabs/kernspark-hono @syncafricabs/kernspark-core hono
```

## Quick Start

```typescript
import { Hono } from 'hono';
import { errors, correlationIdMiddleware, requestLoggingMiddleware } from '@syncafricabs/kernspark-hono';
import { ValidationError, NotFoundError } from '@syncafricabs/kernspark-core';

const app = new Hono();

app.use('*', correlationIdMiddleware());
app.use('*', requestLoggingMiddleware());

app.get('/users/:id', (c) => {
  const user = findUser(c.req.param('id'));
  if (!user) {
    throw new NotFoundError('User not found');
  }
  return response.ok(c, user, 'User retrieved successfully');
});

app.onError(errors());

export default app;
```

## Comprehensive Usage

### Error Handling

The `errors` is Hono error handler middleware that catches `ApplicationError` instances and converts them to standardized JSON responses.

#### Basic Setup

```typescript
import { Hono } from 'hono';
import { errors } from '@syncafricabs/kernspark-hono';

const app = new Hono();

// ... your routes here ...

app.onError(errors());

export default app;
```

#### Custom Error Handler Configuration

```typescript
import { createerrors, errorsOptions } from '@syncafricabs/kernspark-hono';

const options: errorsOptions = {
  logErrors: true,
  logStack: true,
  includeCause: true,
  includeStack: process.env.NODE_ENV === 'development',
  defaultStatusCode: 500,
  logger: (error, c) => {
    console.error({
      correlationId: c.get('correlationId'),
      method: c.req.method,
      url: c.req.url,
      error: error.message,
      stack: error.stack,
    });
  },
};

app.onError(createerrors(options));
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

app.post('/users', async (c) => {
  const body = await c.req.json();
  const { name, email } = body;

  if (!name || !email) {
    throw new MissingFieldsError('Name and email are required');
  }

  if (!isValidEmail(email)) {
    throw new ValidationError('INVALID_EMAIL', 'Email format is invalid');
  }

  if (userExists(email)) {
    throw new ConflictError('User with this email already exists');
  }

  const user = createUser(body);
  return response.created(c, user, 'User created successfully');
});

app.get('/users/:id', (c) => {
  const user = findUser(c.req.param('id'));
  if (!user) {
    throw new NotFoundError('User not found');
  }
  return response.ok(c, user);
});

app.delete('/users/:id', async (c) => {
  const authHeader = c.req.header('Authorization');
  if (!authHeader) {
    throw new InvalidTokenError('Authorization header missing');
  }

  if (!hasPermission(authHeader, 'delete:users')) {
    throw new PermissionDeniedError('You do not have permission to delete users');
  }

  deleteUser(c.req.param('id'));
  return response.noContent(c);
});
```

### Response Adapter

The `response` provides static helper methods for sending responses.

#### Success Responses

```typescript
import { response } from '@syncafricabs/kernspark-hono';

// 200 OK with data
app.get('/users/:id', (c) => {
  const user = findUser(c.req.param('id'));
  response.ok(c, user, 'User retrieved');
});

// 201 Created with data
app.post('/users', async (c) => {
  const user = createUser(await c.req.json());
  response.created(c, user, 'User created');
});

// 204 No Content
app.delete('/users/:id', (c) => {
  deleteUser(c.req.param('id'));
  response.noContent(c);
});

// Send raw ApiSuccess
app.get('/users', (c) => {
  const users = getAllUsers();
  const success = {
    status: 200,
    success: true,
    data: users,
    message: 'Users retrieved',
  };
  return response.sendSuccess(c, success);
});
```

#### Error Responses

```typescript
import { response } from '@syncafricabs/kernspark-hono';

app.post('/login', async (c) => {
  const { email, password } = await c.req.json();

  if (!email || !password) {
    return response.badRequest(c, 'MISSING_FIELDS', 'Email and password are required');
  }

  const token = authenticate(email, password);
  if (!token) {
    return response.unauthorized(c, 'INVALID_CREDENTIALS', 'Invalid email or password');
  }

  return response.ok(c, { token }, 'Login successful');
});
```

### Middleware

#### Correlation ID Middleware

The correlation ID middleware generates or propagates a unique identifier for each request, enabling distributed tracing.

```typescript
import { correlationIdMiddleware, CorrelationIdOptions } from '@syncafricabs/kernspark-hono';

const options: CorrelationIdOptions = {
  headerName: 'x-correlation-id',
  generator: () => crypto.randomUUID(),
};

app.use('*', correlationIdMiddleware(options));

// Later in your code
app.get('/health', (c) => {
  console.log(`Health check with correlation ID: ${c.get('correlationId')}`);
  return response.ok(c, { status: 'healthy' });
});
```

#### Request Logging Middleware

The request logging middleware logs incoming requests and outgoing responses with sanitization.

```typescript
import { requestLoggingMiddleware, RequestLoggingOptions } from '@syncafricabs/kernspark-hono';

const options: RequestLoggingOptions = {
  logRequest: true,
  logResponse: true,
  logBody: true,
  sensitiveHeaders: ['authorization', 'cookie', 'x-api-key'],
  sensitiveBodyFields: ['password', 'token', 'secret', 'apiKey', 'creditCard', 'cvv'],
  logger: (meta) => {
    console.log(JSON.stringify(meta));
  },
};

app.use('*', requestLoggingMiddleware(options));
```

Sample log output:

```json
{
  "method": "POST",
  "url": "/api/users",
  "body": {
    "name": "John Doe",
    "email": "john@example.com",
    "password": "***REDACTED***"
  },
  "correlationId": "550e8400-e29b-41d4-a716-446655440000"
}
```

```json
{
  "method": "POST",
  "url": "/api/users",
  "statusCode": 201,
  "responseTime": 45,
  "correlationId": "550e8400-e29b-41d4-a716-446655440000"
}
```

## API Reference

### `errors`

Hono error handler middleware. Catches all errors and maps them to standardized JSON responses.

```typescript
function errors(error: Error, c: Context): Response | Promise<Response>;
```

### `createerrors(options?: errorsOptions)`

Creates a customized error handler.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `logErrors` | `boolean` | `true` | Enable/disable error logging |
| `logStack` | `boolean` | `false` | Include stack traces in logs |
| `includeCause` | `boolean` | `true` | Include error cause in response |
| `includeStack` | `boolean` | `false` | Include stack traces in response (dev only) |
| `defaultStatusCode` | `number` | `500` | Default status for unknown errors |
| `logger` | `function` | `console.error` | Custom logger function |

### `response`

Static utility class for sending responses.

| Method | HTTP Status | Description |
|--------|-------------|-------------|
| `ok(c, data?, message?)` | 200 | Send success response with data |
| `created(c, data?, message?)` | 201 | Send created response with data |
| `noContent(c, message?)` | 204 | Send no content response |
| `sendSuccess(c, apiSuccess)` | varies | Send raw ApiSuccess |
| `sendError(c, apiError)` | varies | Send raw ApiError |
| `badRequest(c, errorCode, message)` | 400 | Send bad request error |
| `unauthorized(c, errorCode, message)` | 401 | Send unauthorized error |
| `forbidden(c, errorCode, message)` | 403 | Send forbidden error |
| `notFound(c, errorCode, message)` | 404 | Send not found error |
| `conflict(c, errorCode, message)` | 409 | Send conflict error |
| `unprocessableEntity(c, errorCode, message)` | 422 | Send unprocessable entity error |
| `tooManyRequests(c, errorCode, message)` | 429 | Send too many requests error |
| `internalServerError(c, errorCode, message)` | 500 | Send internal server error |
| `badGateway(c, errorCode, message)` | 502 | Send bad gateway error |
| `serviceUnavailable(c, errorCode, message)` | 503 | Send service unavailable error |
| `gatewayTimeout(c, errorCode, message)` | 504 | Send gateway timeout error |

### `correlationIdMiddleware(options?: CorrelationIdOptions)`

Correlation ID middleware.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `headerName` | `string` | `'x-correlation-id'` | Header name for correlation ID |
| `generator` | `function` | `uuidv4` | Generator function for new IDs |

Access correlation ID via `c.get('correlationId')`.

### `requestLoggingMiddleware(options?: RequestLoggingOptions)`

Request/response logging middleware.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `logRequest` | `boolean` | `true` | Log incoming requests |
| `logResponse` | `boolean` | `true` | Log outgoing responses |
| `logBody` | `boolean` | `false` | Log request body |
| `sensitiveHeaders` | `string[]` | `['authorization', 'cookie', 'x-api-key']` | Headers to redact |
| `sensitiveBodyFields` | `string[]` | `['password', 'token', ...]` | Body fields to redact |
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

Full TypeScript support with strict mode enabled.

```typescript
import { Hono } from 'hono';
import { response, correlationIdMiddleware } from '@syncafricabs/kernspark-hono';

type Bindings = {
  DATABASE_URL: string;
};

type Variables = {
  correlationId: string;
};

const app = new Hono<{ Bindings: Bindings; Variables: Variables }>();

app.use('*', correlationIdMiddleware());

app.get('/health', (c) => {
  const correlationId = c.get('correlationId');
  return c.json({ status: 'healthy', correlationId });
});
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

| Package Version | Node.js | Hono | @syncafricabs/kernspark-core |
|----------------|---------|------|----------------------------------|
| 1.0.0 | >=18.0.0 | ^3.0.0 | ^1.0.0 |

| Feature | Hono 3.x |
|---------|----------|
| Error handling | Supported |
| Middleware | Supported |
| Request/response hooks | Supported |
| Context variables | Supported |
| Correlation ID propagation | Supported |
| TypeScript 5.0+ | Supported |
| Cloudflare Workers | Supported |
| Deno | Supported |
| Bun | Supported |
| Node.js | Supported |

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
cd kernspark/packages/kernspark-hono

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
- Add JSDoc comments for public APIs
- Ensure all tests pass before submitting PR

## License

Apache-2.0

## Author

**Providence Chikukwa**
- Email: iamprovy@outlook.com
- GitHub: https://github.com/iamprovy-dev
- LinkedIn: https://www.linkedin.com/in/provychikukwa
- Organization: [SyncAfrica Business Solutions](https://www.syncafricabs.com)


