# @syncafricabs/kernspark-fastify

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue)](https://www.typescriptlang.org/)
[![Fastify](https://img.shields.io/badge/Fastify-4.x-green)](https://www.fastify.io/)

Fastify adapter for the [SyncAfrica KernSpark](https://github.com/iamprovy-dev/kernspark-js) ecosystem. This package provides production-ready Fastify error handlers, response utilities, and plugins that integrate the framework-independent `@syncafricabs/kernspark-core` with Fastify.

## Table of Contents

- [What is it?](#what-is-it)
- [Why it exists](#why-it-exists)
- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Comprehensive Usage](#comprehensive-usage)
  - [Error Handling](#error-handling)
  - [Response Adapter](#response-adapter)
  - [Plugin Registration](#plugin-registration)
- [API Reference](#api-reference)
- [Error Handling Reference](#error-handling-reference)
- [TypeScript Support](#typescript-support)
- [Compatibility Matrix](#compatibility-matrix)
- [Contributing](#contributing)
- [License](#license)

## What is it?

`@syncafricabs/kernspark-fastify` is a framework adapter that bridges the [SyncAfrica KernSpark Core](https://github.com/iamprovy-dev/kernspark-js/tree/main/packages/kernspark-core) with Fastify. It provides:

- **errors** - Fastify error handler that maps `ApplicationError` instances to standardized JSON API responses
- **response** - Helper methods for sending consistent success and error responses
- **plugin** - Fastify plugin that hooks into request/response lifecycle, provides correlation ID propagation, and logging

The adapter follows the **adapter pattern**: it depends on `@syncafricabs/kernspark-core` (the framework-independent layer) and Fastify only in the adapter layer. The core has zero framework dependencies.

## Why it exists

Modern microservice architectures benefit from a **KernSpark** - a common set of contracts, types, and behaviors shared across bounded contexts. The SyncAfrica KernSpark provides:

1. **Standardized API envelopes** (`ApiSuccess` / `ApiError`)
2. **Rich domain error hierarchy** (`ValidationError`, `BusinessError`, `AuthenticationError`, etc.)
3. **Result types** (`Result`, `Ok`, `Err`)
4. **Domain primitives** (`UUID`, `Money`, `Entity`, `ValueObject`)

Without adapters, each team would reimplement framework-specific plumbing to use these core types. This adapter eliminates that duplication by providing Fastify-specific bindings.

## Features

- **Zero-configuration error handling** - Drop-in Fastify error handler
- **Standardized JSON responses** - All responses follow the `ApiSuccess` / `ApiError` envelope
- **Plugin-based architecture** - Register once, works everywhere
- **Correlation ID propagation** - Automatic `x-correlation-id` header management
- **Request/response logging** - Configurable structured logging with sanitization
- **Full TypeScript support** - Comprehensive type definitions included
- **Production-ready** - Proper error handling, stack traces, cause chains
- **Framework isolation** - Core has no Fastify dependencies

## Installation

```bash
npm install @syncafricabs/kernspark-fastify @syncafricabs/kernspark-core fastify
```

Or with your preferred package manager:

```bash
yarn add @syncafricabs/kernspark-fastify @syncafricabs/kernspark-core fastify
```

## Quick Start

```typescript
import Fastify from 'fastify';
import { plugin, errors } from '@syncafricabs/kernspark-fastify';
import { ValidationError, NotFoundError } from '@syncafricabs/kernspark-core';

const app = Fastify({ logger: true });

await app.register(plugin);

app.get('/users/:id', async (request, reply) => {
  const user = findUser(request.params.id);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  return response.ok(reply, user, 'User retrieved successfully');
});

app.setErrorHandler(errors());

await app.listen({ port: 3000 });
```

## Comprehensive Usage

### Error Handling

The `errors` is a Fastify error handler that catches `ApplicationError` instances and converts them to standardized JSON responses.

#### Basic Setup

```typescript
import Fastify from 'fastify';
import { plugin, errors, response } from '@syncafricabs/kernspark-fastify';

const app = Fastify({ logger: true });

await app.register(plugin);

app.get('/users/:id', async (request, reply) => {
  const user = findUser(request.params.id);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  return response.ok(reply, user);
});

app.setErrorHandler(errors());

await app.listen({ port: 3000 });
```

#### Custom Error Handler Configuration

```typescript
import { createerrors, errorsOptions } from '@syncafricabs/kernspark-fastify';

const options: errorsOptions = {
  logErrors: true,
  logStack: true,
  includeCause: true,
  includeStack: process.env.NODE_ENV === 'development',
  defaultStatusCode: 500,
  logger: (error, request) => {
    console.error({
      correlationId: request.correlationId,
      method: request.method,
      url: request.url,
      error: error.message,
      stack: error.stack,
    });
  },
};

app.setErrorHandler(createerrors(options));
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

app.post('/users', async (request, reply) => {
  const { name, email } = request.body as any;

  if (!name || !email) {
    throw new MissingFieldsError('Name and email are required');
  }

  if (!isValidEmail(email)) {
    throw new ValidationError('INVALID_EMAIL', 'Email format is invalid');
  }

  if (userExists(email)) {
    throw new ConflictError('User with this email already exists');
  }

  const user = createUser(request.body);
  return response.created(reply, user, 'User created successfully');
});

app.get('/users/:id', async (request, reply) => {
  const user = findUser(request.params.id);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  return response.ok(reply, user);
});

app.delete('/users/:id', async (request, reply) => {
  if (!request.user) {
    throw new InvalidTokenError('Invalid or expired token');
  }

  if (!request.user.canDeleteUsers) {
    throw new PermissionDeniedError('You do not have permission to delete users');
  }

  deleteUser(request.params.id);
  return response.noContent(reply);
});
```

### Response Adapter

The `response` provides static helper methods for sending responses.

#### Success Responses

```typescript
import { response } from '@syncafricabs/kernspark-fastify';

// 200 OK with data
app.get('/users/:id', async (request, reply) => {
  const user = findUser(request.params.id);
  response.ok(reply, user, 'User retrieved');
});

// 201 Created with data
app.post('/users', async (request, reply) => {
  const user = createUser(request.body);
  response.created(reply, user, 'User created');
});

// 204 No Content
app.delete('/users/:id', async (request, reply) => {
  deleteUser(request.params.id);
  response.noContent(reply);
});

// Send raw ApiSuccess
app.get('/users', async (request, reply) => {
  const users = getAllUsers();
  const success = {
    status: 200,
    success: true,
    data: users,
    message: 'Users retrieved',
  };
  response.sendSuccess(reply, success);
});
```

#### Error Responses

```typescript
import { response } from '@syncafricabs/kernspark-fastify';

app.post('/login', async (request, reply) => {
  const { email, password } = request.body as any;

  if (!email || !password) {
    return response.badRequest(reply, 'MISSING_FIELDS', 'Email and password are required');
  }

  const token = authenticate(email, password);
  if (!token) {
    return response.unauthorized(reply, 'INVALID_CREDENTIALS', 'Invalid email or password');
  }

  return response.ok(reply, { token }, 'Login successful');
});
```

### Plugin Registration

The `plugin` is a Fastify plugin that registers hooks and decorators for correlation ID, request/response logging, and global error handling.

#### Basic Plugin Registration

```typescript
import Fastify from 'fastify';
import { plugin } from '@syncafricabs/kernspark-fastify';

const app = Fastify({ logger: true });

await app.register(plugin);

// Now correlationId is available on all requests
app.get('/health', async (request, reply) => {
  console.log(`Health check with correlation ID: ${request.correlationId}`);
  return { status: 'healthy' };
});
```

#### Custom Plugin Options

```typescript
import { plugin, FastifyPluginOptions } from '@syncafricabs/kernspark-fastify';

const options: FastifyPluginOptions = {
  correlationIdHeader: 'x-correlation-id',
  logRequests: true,
  logResponses: true,
  logErrors: true,
  sensitiveFields: ['password', 'token', 'secret', 'apiKey', 'creditCard', 'cvv'],
  logger: (meta) => {
    console.log(JSON.stringify(meta));
  },
};

await app.register(plugin, options);
```

#### Plugin with Error Handler

```typescript
import Fastify from 'fastify';
import { plugin, errors, response } from '@syncafricabs/kernspark-fastify';

const app = Fastify({ logger: true });

await app.register(plugin, {
  correlationIdHeader: 'x-request-id',
  logRequests: true,
  logResponses: true,
  sensitiveFields: ['password', 'secret'],
});

app.get('/users/:id', async (request, reply) => {
  const user = findUser(request.params.id);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  return response.ok(reply, user);
});

app.setErrorHandler(errors({
  logErrors: true,
  includeStack: process.env.NODE_ENV === 'development',
}));

await app.listen({ port: 3000 });
```

## API Reference

### `errors`

Fastify error handler function. Catches all errors and maps them to standardized JSON responses.

```typescript
function errors(options?: errorsOptions): (error: Error, request: FastifyRequest, reply: FastifyReply) => Promise<void>;
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
| `ok(reply, data?, message?)` | 200 | Send success response with data |
| `created(reply, data?, message?)` | 201 | Send created response with data |
| `noContent(reply, message?)` | 204 | Send no content response |
| `sendSuccess(reply, apiSuccess)` | varies | Send raw ApiSuccess |
| `sendError(reply, apiError)` | varies | Send raw ApiError |
| `badRequest(reply, errorCode, message)` | 400 | Send bad request error |
| `unauthorized(reply, errorCode, message)` | 401 | Send unauthorized error |
| `forbidden(reply, errorCode, message)` | 403 | Send forbidden error |
| `notFound(reply, errorCode, message)` | 404 | Send not found error |
| `conflict(reply, errorCode, message)` | 409 | Send conflict error |
| `unprocessableEntity(reply, errorCode, message)` | 422 | Send unprocessable entity error |
| `tooManyRequests(reply, errorCode, message)` | 429 | Send too many requests error |
| `internalServerError(reply, errorCode, message)` | 500 | Send internal server error |
| `badGateway(reply, errorCode, message)` | 502 | Send bad gateway error |
| `serviceUnavailable(reply, errorCode, message)` | 503 | Send service unavailable error |
| `gatewayTimeout(reply, errorCode, message)` | 504 | Send gateway timeout error |

### `plugin`

Fastify plugin for KernSpark functionality.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `correlationIdHeader` | `string` | `'x-correlation-id'` | Header name for correlation ID |
| `logRequests` | `boolean` | `true` | Log incoming requests |
| `logResponses` | `boolean` | `true` | Log outgoing responses |
| `logErrors` | `boolean` | `true` | Log errors |
| `sensitiveFields` | `string[]` | `['password', 'token', ...]` | Body fields to redact |
| `logger` | `function` | `console.info` | Custom logger function |

**Decorators added:**
- `request.correlationId` - Correlation ID for the request
- `request.startTime` - Request start timestamp

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

Full TypeScript support with strict mode enabled. The adapter provides type augmentations for Fastify:

```typescript
import 'fastify';
import { FastifyRequest, FastifyReply } from 'fastify';

// With the plugin registered, request.correlationId is typed
const handler = async (request: FastifyRequest, reply: FastifyReply) => {
  const correlationId = request.correlationId; // string
  console.log(`Request with correlation ID: ${correlationId}`);
};
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

| Package Version | Node.js | Fastify | @syncafricabs/kernspark-core |
|----------------|---------|---------|----------------------------------|
| 1.0.0 | >=18.0.0 | ^4.0.0 | ^1.0.0 |

| Feature | Fastify 4.x |
|---------|-------------|
| Error handler | Supported |
| Plugin registration | Supported |
| Request/response hooks | Supported |
| Decorators | Supported |
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
cd kernspark/packages/kernspark-fastify

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


