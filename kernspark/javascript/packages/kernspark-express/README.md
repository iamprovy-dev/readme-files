# @syncafricabs/kernspark-express

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue)](https://www.typescriptlang.org/)
[![Express](https://img.shields.io/badge/Express-4.x-green)](https://expressjs.com/)

Express adapter for the [SyncAfrica KernSpark](https://github.com/iamprovy-dev/kernspark-js) ecosystem. This package provides production-ready Express middleware, error handlers, and response utilities that integrate the framework-independent `@syncafricabs/kernspark-core` with Express.js.

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

`@syncafricabs/kernspark-express` is a framework adapter that bridges the [SyncAfrica KernSpark Core](https://github.com/iamprovy-dev/kernspark-js/tree/main/packages/kernspark-core) with Express.js. It provides:

- **errors** - Maps `ApplicationError` instances to standardized JSON API responses
- **response** - Helper methods for sending consistent success and error responses
- **middleware** - Request/response logging and correlation ID tracking

The adapter follows the **adapter pattern**: it depends on `@syncafricabs/kernspark-core` (the framework-independent layer) and Express only in the adapter layer. The core has zero framework dependencies.

## Why it exists

Modern microservice architectures benefit from a **KernSpark** - a common set of contracts, types, and behaviors shared across bounded contexts. The SyncAfrica KernSpark provides:

1. **Standardized API envelopes** (`ApiSuccess` / `ApiError`)
2. **Rich domain error hierarchy** (`ValidationError`, `BusinessError`, `AuthenticationError`, etc.)
3. **Result types** (`Result`, `Ok`, `Err`)
4. **Domain primitives** (`UUID`, `Money`, `Entity`, `ValueObject`)

Without adapters, each team would reimplement framework-specific plumbing to use these core types. This adapter eliminates that duplication by providing Express-specific bindings.

## Features

- **Zero-configuration error handling** - Drop-in Express error middleware
- **Standardized JSON responses** - All responses follow the `ApiSuccess` / `ApiError` envelope
- **Correlation ID propagation** - Automatic `x-correlation-id` header management
- **Request/response logging** - Configurable structured logging with sanitization
- **Full TypeScript support** - Comprehensive type definitions included
- **Production-ready** - Proper error handling, stack traces, cause chains
- **Framework isolation** - Core has no Express dependencies

## Installation

```bash
npm install @syncafricabs/kernspark-express @syncafricabs/kernspark-core express
```

Or with your preferred package manager:

```bash
yarn add @syncafricabs/kernspark-express @syncafricabs/kernspark-core express
```

```bash
pnpm add @syncafricabs/kernspark-express @syncafricabs/kernspark-core express
```

## Quick Start

```typescript
import express from 'express';
import { errors, correlationIdMiddleware, requestLoggingMiddleware } from '@syncafricabs/kernspark-express';
import { ValidationError, NotFoundError } from '@syncafricabs/kernspark-core';

const app = express();

app.use(express.json());
app.use(correlationIdMiddleware());
app.use(requestLoggingMiddleware());

app.get('/users/:id', (req, res) => {
  const user = findUser(req.params.id);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  return response.ok(res, user, 'User retrieved successfully');
});

app.use(errors());

app.listen(3000, () => console.log('Server running on port 3000'));
```

## Comprehensive Usage

### Error Handling

The `errors` default instance is an Express error-handling middleware that catches `ApplicationError` instances and converts them to standardized JSON responses.

#### Basic Setup

```typescript
import express from 'express';
import { errors } from '@syncafricabs/kernspark-express';

const app = express();

// ... your routes here ...

// Error handler must be registered AFTER all routes
app.use(errors());
```

#### Custom Error Handler Configuration

```typescript
import { createerrors, errorsOptions } from '@syncafricabs/kernspark-express';

const options: errorsOptions = {
  logErrors: true,
  logStack: true,
  includeCause: true,
  includeStack: process.env.NODE_ENV === 'development',
  defaultStatusCode: 500,
  logger: (error, req) => {
    console.error({
      correlationId: req.correlationId,
      method: req.method,
      url: req.originalUrl,
      error: error.message,
      stack: error.stack,
    });
  },
};

app.use(createerrors(options));
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

app.post('/users', (req, res, next) => {
  const { name, email } = req.body;

  if (!name || !email) {
    throw new MissingFieldsError('Name and email are required');
  }

  if (!isValidEmail(email)) {
    throw new ValidationError('INVALID_EMAIL', 'Email format is invalid');
  }

  if (userExists(email)) {
    throw new ConflictError('User with this email already exists');
  }

  const user = createUser(req.body);
  return response.created(res, user, 'User created successfully');
});

app.get('/users/:id', (req, res, next) => {
  const user = findUser(req.params.id);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  return response.ok(res, user);
});

app.delete('/users/:id', (req, res, next) => {
  if (!req.user) {
    throw new InvalidTokenError('Invalid or expired token');
  }

  if (!req.user.canDeleteUsers) {
    throw new PermissionDeniedError('You do not have permission to delete users');
  }

  deleteUser(req.params.id);
  return response.noContent(res);
});
```

#### Catching Async Errors

Express 5 has built-in async error handling, but for Express 4 you need a wrapper:

```typescript
import { Request, Response, NextFunction } from 'express';

const asyncHandler = (fn: (req: Request, res: Response, next: NextFunction) => Promise<any>) => {
  return (req: Request, res: Response, next: NextFunction) => {
    fn(req, res, next).catch(next);
  };
};

app.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await findUserAsync(req.params.id);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  return response.ok(res, user);
}));
```

### Response Adapter

The `response` provides static helper methods for sending consistent success and error responses.

#### Success Responses

```typescript
import { response } from '@syncafricabs/kernspark-express';

// 200 OK with data
app.get('/users/:id', (req, res) => {
  const user = findUser(req.params.id);
  response.ok(res, user, 'User retrieved');
});

// 201 Created with data
app.post('/users', (req, res) => {
  const user = createUser(req.body);
  response.created(res, user, 'User created');
});

// 204 No Content
app.delete('/users/:id', (req, res) => {
  deleteUser(req.params.id);
  response.noContent(res);
});

// Send raw ApiSuccess
app.get('/users', (req, res) => {
  const users = getAllUsers();
  const success = {
    status: 200,
    success: true,
    data: users,
    message: 'Users retrieved',
  };
  response.sendSuccess(res, success);
});
```

#### Error Responses

```typescript
import { response } from '@syncafricabs/kernspark-express';

app.post('/login', (req, res) => {
  const { email, password } = req.body;

  if (!email || !password) {
    return response.badRequest(res, 'MISSING_FIELDS', 'Email and password are required');
  }

  const token = authenticate(email, password);
  if (!token) {
    return response.unauthorized(res, 'INVALID_CREDENTIALS', 'Invalid email or password');
  }

  return response.ok(res, { token }, 'Login successful');
});
```

### Middleware

#### Correlation ID Middleware

The correlation ID middleware generates or propagates a unique identifier for each request, enabling distributed tracing.

```typescript
import { correlationIdMiddleware, CorrelationIdOptions } from '@syncafricabs/kernspark-express';

const options: CorrelationIdOptions = {
  headerName: 'x-correlation-id',
  generator: () => crypto.randomUUID(),
};

app.use(correlationIdMiddleware(options));

// Later in your code
app.get('/health', (req, res) => {
  console.log(`Health check with correlation ID: ${req.correlationId}`);
  response.ok(res, { status: 'healthy' });
});
```

#### Request Logging Middleware

The request logging middleware logs incoming requests and outgoing responses with sanitization.

```typescript
import { requestLoggingMiddleware, RequestLoggingOptions } from '@syncafricabs/kernspark-express';

const options: RequestLoggingOptions = {
  logRequest: true,
  logResponse: true,
  logBody: true,
  logQuery: true,
  sensitiveHeaders: ['authorization', 'cookie', 'x-api-key'],
  sensitiveBodyFields: ['password', 'token', 'secret', 'apiKey', 'creditCard', 'cvv'],
  logger: (meta) => {
    console.log(JSON.stringify(meta));
  },
};

app.use(requestLoggingMiddleware(options));
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
  "contentLength": 156,
  "responseTime": 45,
  "correlationId": "550e8400-e29b-41d4-a716-446655440000"
}
```

## API Reference

### `errors`

Express error handling middleware. Catches all errors and maps them to standardized JSON responses.

```typescript
function errors(error: Error, req: Request, res: Response, next: NextFunction): void;
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
| `ok(res, data?, message?)` | 200 | Send success response with data |
| `created(res, data?, message?)` | 201 | Send created response with data |
| `noContent(res, message?)` | 204 | Send no content response |
| `sendSuccess(res, apiSuccess)` | varies | Send raw ApiSuccess |
| `sendError(res, apiError)` | varies | Send raw ApiError |
| `badRequest(res, errorCode, message)` | 400 | Send bad request error |
| `unauthorized(res, errorCode, message)` | 401 | Send unauthorized error |
| `forbidden(res, errorCode, message)` | 403 | Send forbidden error |
| `notFound(res, errorCode, message)` | 404 | Send not found error |
| `conflict(res, errorCode, message)` | 409 | Send conflict error |
| `unprocessableEntity(res, errorCode, message)` | 422 | Send unprocessable entity error |
| `tooManyRequests(res, errorCode, message)` | 429 | Send too many requests error |
| `internalServerError(res, errorCode, message)` | 500 | Send internal server error |
| `badGateway(res, errorCode, message)` | 502 | Send bad gateway error |
| `serviceUnavailable(res, errorCode, message)` | 503 | Send service unavailable error |
| `gatewayTimeout(res, errorCode, message)` | 504 | Send gateway timeout error |

### `correlationIdMiddleware(options?: CorrelationIdOptions)`

Correlation ID middleware.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `headerName` | `string` | `'x-correlation-id'` | Header name for correlation ID |
| `generator` | `function` | `uuidv4` | Generator function for new IDs |

Attaches `req.correlationId` and sets response header.

### `requestLoggingMiddleware(options?: RequestLoggingOptions)`

Request/response logging middleware.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `logRequest` | `boolean` | `true` | Log incoming requests |
| `logResponse` | `boolean` | `true` | Log outgoing responses |
| `logBody` | `boolean` | `false` | Log request body |
| `logQuery` | `boolean` | `false` | Log query parameters |
| `sensitiveHeaders` | `string[]` | `['authorization', 'cookie', 'set-cookie']` | Headers to redact |
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

Full TypeScript support with strict mode enabled. The adapter extends Express types:

```typescript
import { Request } from 'express';

// correlationId is added by the middleware
interface RequestWithCorrelationId extends Request {
  correlationId: string;
}

// Express error handler type augmentation is provided via module declaration
declare module 'express' {
  interface Request {
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

| Package Version | Node.js | Express | @syncafricabs/kernspark-core |
|----------------|---------|---------|----------------------------------|
| 1.0.0 | >=18.0.0 | ^4.18.0 | ^1.0.0 |

| Feature | Express 4.18+ |
|---------|---------------|
| Error handling middleware | Supported |
| Request/response logging | Supported |
| Correlation ID propagation | Supported |
| Async/await | Supported (with wrapper) |
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
cd kernspark/packages/kernspark-express

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


