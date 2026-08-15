# @syncafricabs/kernspark-sailsjs

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue)](https://www.typescriptlang.org/)
[![SailsJS](https://img.shields.io/badge/SailsJS-1.5%2B-DE0982)](https://sailsjs.com/)

SailsJS adapter for the [SyncAfrica KernSpark](https://github.com/iamprovy-dev/kernspark-js) ecosystem. This package provides production-ready SailsJS response interceptors, error handlers, and utilities that integrate the framework-independent `@syncafricabs/kernspark-core` with SailsJS.

## Table of Contents

- [What is it?](#what-is-it)
- [Why it exists](#why-it-exists)
- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Comprehensive Usage](#comprehensive-usage)
  - [Response Interceptors](#response-interceptors)
  - [Error Handling](#error-handling)
  - [Response Adapter](#response-adapter)
  - [Complete Controller Example](#complete-controller-example)
- [API Reference](#api-reference)
- [Error Handling Reference](#error-handling-reference)
- [TypeScript Support](#typescript-support)
- [Compatibility Matrix](#compatibility-matrix)
- [Contributing](#contributing)
- [License](#license)
- [Author](#author)

## What is it?

`@syncafricabs/kernspark-sailsjs` is a framework adapter that bridges the [SyncAfrica KernSpark Core](https://github.com/iamprovy-dev/kernspark-js/tree/main/packages/kernspark-core) with SailsJS. It provides:

- **SailsJsIntegration** - Maps `ApplicationError` instances to standardized JSON API responses
- **SailsJsResponseAdapter** - Helper methods for sending consistent success and error responses
- **sailsJsErrorHandler** - Pre-configured error handler for SailsJS
- **withSailsErrorHandling** - Higher-order function for wrapping controllers with error handling

The adapter follows the **adapter pattern**: it depends on `@syncafricabs/kernspark-core` (the framework-independent layer) and SailsJS only in the adapter layer. The core has zero framework dependencies.

## Why it exists

Modern web applications benefit from a **KernSpark** - a common set of contracts, types, and behaviors shared across bounded contexts. The SyncAfrica KernSpark provides:

1. **Standardized API envelopes** (`ApiSuccess` / `ApiError`)
2. **Rich domain error hierarchy** (`ValidationError`, `BusinessError`, `AuthenticationError`, etc.)
3. **Result types** (`Result`, `Ok`, `Err`)
4. **Domain primitives** (`UUID`, `Money`, `Entity`, `ValueObject`)

Without adapters, each team would reimplement framework-specific plumbing to use these core types. This adapter eliminates that duplication by providing SailsJS-specific bindings.

## Features

- **Response interceptor support** - Drop-in SailsJS response interceptor
- **Standardized JSON responses** - All responses follow the `ApiSuccess` / `ApiError` envelope
- **Zero-configuration error handling** - Pre-configured error handler
- **Full TypeScript support** - Comprehensive type definitions included
- **Production-ready** - Proper error handling, stack traces, cause chains
- **Framework isolation** - Core has no SailsJS dependencies

## Installation

```bash
npm install @syncafricabs/kernspark-sailsjs @syncafricabs/kernspark-core sails
```

Or with your preferred package manager:

```bash
yarn add @syncafricabs/kernspark-sailsjs @syncafricabs/kernspark-core sails
```

```bash
pnpm add @syncafricabs/kernspark-sailsjs @syncafricabs/kernspark-core sails
```

## Quick Start

```typescript
// api/controllers/UserController.ts
import type { Request, Response } from 'express';
import { withSailsErrorHandling, SailsJsResponseAdapter } from '@syncafricabs/kernspark-sailsjs';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

export default class UserController {
  async find(req: Request, res: Response) {
    const users = await User.find();
    SailsJsResponseAdapter.ok(res, users, 'Users retrieved successfully');
  }

  async findOne(req: Request, res: Response) {
    const user = await User.findOne({ id: req.params.id });
    if (!user) {
      throw new NotFoundError('User not found');
    }
    SailsJsResponseAdapter.ok(res, user, 'User retrieved successfully');
  }

  async create(req: Request, res: Response) {
    if (!req.body.name || !req.body.email) {
      throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
    }
    const user = await User.create(req.body);
    SailsJsResponseAdapter.created(res, user, 'User created successfully');
  }
}
```

## Comprehensive Usage

### Response Interceptors

#### Basic Setup

```typescript
// config/http.js
module.exports.http = {
  middleware: {
    order: [
      'cookieParser',
      'session',
      'bodyParser',
      'compress',
      'poweredBy',
      'router',
      'www',
      'favicon',
    ],
  },
};
```

```typescript
// api/responses/kernspark.ts
import { sailsJsErrorHandler } from '@syncafricabs/kernspark-sailsjs';

export default sailsJsErrorHandler;
```

```typescript
// config/http.js
module.exports.http = {
  middleware: {
    order: [
      'cookieParser',
      'session',
      'bodyParser',
      'compress',
      'poweredBy',
      'router',
      'www',
      'favicon',
      'kernspark',
    ],
  },
};
```

### Error Handling

#### Custom Error Handler Configuration

```typescript
import { createSailsJsErrorHandler, SailsJsErrorHandlerOptions } from '@syncafricabs/kernspark-sailsjs';

const options: SailsJsErrorHandlerOptions = {
  logErrors: true,
  logStack: true,
  includeCause: true,
  includeStack: process.env.NODE_ENV === 'development',
  defaultStatusCode: 500,
  logger: (error, req) => {
    console.error({
      method: req.method,
      url: req.originalUrl || req.url,
      error: error.message,
      stack: error.stack,
    });
  },
};

module.exports.http = {
  middleware: {
    order: [
      'cookieParser',
      'session',
      'bodyParser',
      'compress',
      'poweredBy',
      'router',
      'www',
      'favicon',
      'kernspark',
    ],
  },
};
```

### Response Adapter

#### Success Responses

```typescript
import { SailsJsResponseAdapter } from '@syncafricabs/kernspark-sailsjs';

// 200 OK with data
SailsJsResponseAdapter.ok(res, user, 'User retrieved');

// 201 Created with data
SailsJsResponseAdapter.created(res, user, 'User created');

// 204 No Content
SailsJsResponseAdapter.noContent(res);
```

#### Error Responses

```typescript
import { SailsJsResponseAdapter } from '@syncafricabs/kernspark-sailsjs';

SailsJsResponseAdapter.badRequest(res, 'MISSING_FIELDS', 'Email and password are required');
SailsJsResponseAdapter.unauthorized(res, 'INVALID_CREDENTIALS', 'Invalid email or password');
SailsJsResponseAdapter.forbidden(res, 'PERMISSION_DENIED', 'You do not have permission');
SailsJsResponseAdapter.notFound(res, 'NOT_FOUND', 'User not found');
SailsJsResponseAdapter.conflict(res, 'CONFLICT', 'Resource already exists');
SailsJsResponseAdapter.unprocessableEntity(res, 'INVALID_DATA', 'Invalid request data');
SailsJsResponseAdapter.tooManyRequests(res, 'RATE_LIMIT_EXCEEDED', 'Too many requests');
SailsJsResponseAdapter.internalServerError(res, 'INTERNAL_ERROR', 'An unexpected error occurred');
SailsJsResponseAdapter.badGateway(res, 'BAD_GATEWAY', 'Upstream service error');
SailsJsResponseAdapter.serviceUnavailable(res, 'SERVICE_UNAVAILABLE', 'Service temporarily unavailable');
SailsJsResponseAdapter.gatewayTimeout(res, 'GATEWAY_TIMEOUT', 'Upstream service timeout');
```

### Complete Controller Example

```typescript
// api/controllers/UserController.ts
import type { Request, Response } from 'express';
import { withSailsErrorHandling, SailsJsResponseAdapter } from '@syncafricabs/kernspark-sailsjs';
import { NotFoundError, ValidationError, ConflictError, InvalidTokenError, PermissionDeniedError } from '@syncafricabs/kernspark-core';

class UserController {
  async find(req: Request, res: Response) {
    const users = await User.find();
    SailsJsResponseAdapter.ok(res, users, 'Users retrieved successfully');
  }

  async findOne(req: Request, res: Response) {
    const user = await User.findOne({ id: req.params.id });
    if (!user) {
      throw new NotFoundError('User not found');
    }
    SailsJsResponseAdapter.ok(res, user, 'User retrieved successfully');
  }

  async create(req: Request, res: Response) {
    const { name, email } = req.body;

    if (!name || !email) {
      throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
    }

    if (await User.findOne({ email })) {
      throw new ConflictError('User with this email already exists');
    }

    const user = await User.create({ name, email });
    SailsJsResponseAdapter.created(res, user, 'User created successfully');
  }

  async update(req: Request, res: Response) {
    const user = await User.findOne({ id: req.params.id });
    if (!user) {
      throw new NotFoundError('User not found');
    }

    const updated = await User.updateOne({ id: req.params.id }).set(req.body);
    SailsJsResponseAdapter.ok(res, updated, 'User updated successfully');
  }

  async delete(req: Request, res: Response) {
    const token = req.headers.authorization;
    if (!token || !isValidToken(token)) {
      throw new InvalidTokenError('Invalid or expired token');
    }

    const user = await User.findOne({ id: req.params.id });
    if (!user) {
      throw new NotFoundError('User not found');
    }

    if (!canDeleteUser(token, user.id)) {
      throw new PermissionDeniedError('You do not have permission to delete users');
    }

    await User.destroyOne(user.id);
    SailsJsResponseAdapter.noContent(res, 'User deleted successfully');
  }
}

export default withSailsErrorHandling(new UserController());
```

## API Reference

### `sailsJsErrorHandler`

SailsJS error handling handler. Catches all errors and maps them to standardized JSON responses.

```typescript
function sailsJsErrorHandler(error: Error, req: Request, res: Response, next: NextFunction): void;
```

### `createSailsJsErrorHandler(options?: SailsJsErrorHandlerOptions)`

Creates a customized error handler.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `logErrors` | `boolean` | `true` | Enable/disable error logging |
| `logStack` | `boolean` | `false` | Include stack traces in logs |
| `includeCause` | `boolean` | `true` | Include error cause in response |
| `includeStack` | `boolean` | `false` | Include stack traces in response (dev only) |
| `defaultStatusCode` | `number` | `500` | Default status for unknown errors |
| `logger` | `function` | `console.error` | Custom logger function |

### `SailsJsResponseAdapter`

Static utility class for sending responses.

| Method | HTTP Status | Description |
|--------|-------------|-------------|
| `sendSuccess(res, apiSuccess)` | varies | Send raw ApiSuccess |
| `sendError(res, apiError)` | varies | Send raw ApiError |
| `ok(res, data?, message?)` | 200 | Send success response with data |
| `created(res, data?, message?)` | 201 | Send created response with data |
| `noContent(res, message?)` | 204 | Send no content response |
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

### `withSailsErrorHandling(controller)`

Wraps a SailsJS controller with error handling.

```typescript
function withSailsErrorHandling<T extends SailsJsController>(
  controller: T
): SailsJsController;
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

Full TypeScript support with strict mode enabled. The adapter extends SailsJS/Express types:

```typescript
import type { Request, Response, NextFunction } from 'express';

// Request type augmentation
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

| Package Version | Node.js | SailsJS | @syncafricabs/kernspark-core |
|----------------|---------|---------|----------------------------------|
| 1.0.0 | >=18.0.0 | ^1.5.0 | ^1.0.0 |

| Feature | SailsJS 1.5+ |
|---------|-------------|
| Response interceptors | Supported |
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
cd kernspark/packages/kernspark-sailsjs

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


