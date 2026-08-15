# @syncafricabs/kernspark-strapi

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue)](https://www.typescriptlang.org/)
[![Strapi](https://img.shields.io/badge/Strapi-4%2B-2F68C4)](https://strapi.io/)

Strapi adapter for the [SyncAfrica KernSpark](https://github.com/iamprovy-dev/kernspark-js) ecosystem. This package provides production-ready Strapi lifecycle middleware, error handlers, and utilities that integrate the framework-independent `@syncafricabs/kernspark-core` with Strapi.

## Table of Contents

- [What is it?](#what-is-it)
- [Why it exists](#why-it-exists)
- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Comprehensive Usage](#comprehensive-usage)
  - [Lifecycle Middleware](#lifecycle-middleware)
  - [Error Handling](#error-handling)
  - [Response Adapter](#response-adapter)
  - [Complete Lifecycle Example](#complete-lifecycle-example)
- [API Reference](#api-reference)
- [Error Handling Reference](#error-handling-reference)
- [TypeScript Support](#typescript-support)
- [Compatibility Matrix](#compatibility-matrix)
- [Contributing](#contributing)
- [License](#license)
- [Author](#author)

## What is it?

`@syncafricabs/kernspark-strapi` is a framework adapter that bridges the [SyncAfrica KernSpark Core](https://github.com/iamprovy-dev/kernspark-js/tree/main/packages/kernspark-core) with Strapi. It provides:

- **StrapiIntegration** - Maps `ApplicationError` instances to standardized JSON API responses
- **StrapiResponseAdapter** - Helper methods for sending consistent success and error responses
- **withStrapiErrorHandling** - Higher-order function for wrapping lifecycle middleware with error handling
- **Lifecycle hooks** - Pre-configured before/after create, update, and delete hooks

The adapter follows the **adapter pattern**: it depends on `@syncafricabs/kernspark-core` (the framework-independent layer) and Strapi only in the adapter layer. The core has zero framework dependencies.

## Why it exists

Modern web applications benefit from a **KernSpark** - a common set of contracts, types, and behaviors shared across bounded contexts. The SyncAfrica KernSpark provides:

1. **Standardized API envelopes** (`ApiSuccess` / `ApiError`)
2. **Rich domain error hierarchy** (`ValidationError`, `BusinessError`, `AuthenticationError`, etc.)
3. **Result types** (`Result`, `Ok`, `Err`)
4. **Domain primitives** (`UUID`, `Money`, `Entity`, `ValueObject`)

Without adapters, each team would reimplement framework-specific plumbing to use these core types. This adapter eliminates that duplication by providing Strapi-specific bindings.

## Features

- **Lifecycle middleware support** - Drop-in Strapi lifecycle error handling
- **Standardized JSON responses** - All responses follow the `ApiSuccess` / `ApiError` envelope
- **Zero-configuration error handling** - Wraps lifecycle middleware automatically
- **Full TypeScript support** - Comprehensive type definitions included
- **Production-ready** - Proper error handling, stack traces, cause chains
- **Framework isolation** - Core has no Strapi dependencies

## Installation

```bash
npm install @syncafricabs/kernspark-strapi @syncafricabs/kernspark-core @strapi/strapi
```

Or with your preferred package manager:

```bash
yarn add @syncafricabs/kernspark-strapi @syncafricabs/kernspark-core @strapi/strapi
```

```bash
pnpm add @syncafricabs/kernspark-strapi @syncafricabs/kernspark-core @strapi/strapi
```

## Quick Start

```typescript
// src/api/user/content-types/user/lifecycles.ts
import { withStrapiErrorHandling, StrapiResponseAdapter } from '@syncafricabs/kernspark-strapi';
import { ValidationError, NotFoundError } from '@syncafricabs/kernspark-core';

export default {
  async beforeCreate(event) {
    const { data } = event;

    if (!data.name || !data.email) {
      throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
    }
  },

  async afterCreate(event) {
    if (event.result) {
      event.result = new (require('@syncafricabs/kernspark-core').ApiSuccess)('Created', event.result);
    }
  },
};
```

## Comprehensive Usage

### Lifecycle Middleware

#### Before Create Hook

```typescript
// src/api/user/content-types/user/lifecycles.ts
import { withStrapiErrorHandling, StrapiResponseAdapter } from '@syncafricabs/kernspark-strapi';
import { ValidationError, ConflictError } from '@syncafricabs/kernspark-core';

export default {
  async beforeCreate(event) {
    const { data } = event;

    if (!data.name || !data.email) {
      throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
    }

    if (!isValidEmail(data.email)) {
      throw new ValidationError('INVALID_EMAIL', 'Email format is invalid');
    }

    const existingUser = await strapi.entityService.findMany('api::user.user', {
      filters: { email: data.email },
    });

    if (existingUser.length > 0) {
      throw new ConflictError('User with this email already exists');
    }
  },
};
```

#### After Create Hook

```typescript
// src/api/user/content-types/user/lifecycles.ts
import { strapiAfterCreateHook } from '@syncafricabs/kernspark-strapi';

export default {
  async afterCreate(event) {
    strapiAfterCreateHook(event);
  },
};
```

#### Before Update Hook

```typescript
// src/api/user/content-types/user/lifecycles.ts
import { withStrapiErrorHandling } from '@syncafricabs/kernspark-strapi';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

export default {
  async beforeUpdate(event) {
    const { data, params } = event;

    if (params.where && params.where.id) {
      const user = await strapi.entityService.findOne('api::user.user', params.where.id);
      if (!user) {
        throw new NotFoundError('User not found');
      }
    }

    if (data.email && !isValidEmail(data.email)) {
      throw new ValidationError('INVALID_EMAIL', 'Email format is invalid');
    }
  },
};
```

### Error Handling

#### Custom Error Handler Configuration

```typescript
import { createStrapiErrorHandler, StrapiErrorHandlerOptions } from '@syncafricabs/kernspark-strapi';

const options: StrapiErrorHandlerOptions = {
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
```

### Response Adapter

#### Success Responses

```typescript
import { StrapiResponseAdapter } from '@syncafricabs/kernspark-strapi';

// 200 OK with data
StrapiResponseAdapter.ok(response, user, 'User retrieved');

// 201 Created with data
StrapiResponseAdapter.created(response, user, 'User created');

// 204 No Content
StrapiResponseAdapter.noContent(response);
```

#### Error Responses

```typescript
import { StrapiResponseAdapter } from '@syncafricabs/kernspark-strapi';

StrapiResponseAdapter.badRequest(response, 'MISSING_FIELDS', 'Email and password are required');
StrapiResponseAdapter.unauthorized(response, 'INVALID_CREDENTIALS', 'Invalid email or password');
StrapiResponseAdapter.forbidden(response, 'PERMISSION_DENIED', 'You do not have permission');
StrapiResponseAdapter.notFound(response, 'NOT_FOUND', 'User not found');
StrapiResponseAdapter.conflict(response, 'CONFLICT', 'Resource already exists');
StrapiResponseAdapter.unprocessableEntity(response, 'INVALID_DATA', 'Invalid request data');
StrapiResponseAdapter.tooManyRequests(response, 'RATE_LIMIT_EXCEEDED', 'Too many requests');
StrapiResponseAdapter.internalServerError(response, 'INTERNAL_ERROR', 'An unexpected error occurred');
StrapiResponseAdapter.badGateway(response, 'BAD_GATEWAY', 'Upstream service error');
StrapiResponseAdapter.serviceUnavailable(response, 'SERVICE_UNAVAILABLE', 'Service temporarily unavailable');
StrapiResponseAdapter.gatewayTimeout(response, 'GATEWAY_TIMEOUT', 'Upstream service timeout');
```

### Complete Lifecycle Example

```typescript
// src/api/user/content-types/user/lifecycles.ts
import {
  withStrapiErrorHandling,
  StrapiResponseAdapter,
  strapiBeforeCreateHook,
  strapiAfterCreateHook,
  strapiBeforeUpdateHook,
  strapiAfterUpdateHook,
  strapiBeforeDeleteHook,
  strapiAfterDeleteHook,
} from '@syncafricabs/kernspark-strapi';
import { ValidationError, NotFoundError, ConflictError, InvalidTokenError, PermissionDeniedError } from '@syncafricabs/kernspark-core';

export default {
  async beforeCreate(event) {
    const { data } = event;

    if (!data.name || !data.email) {
      throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
    }

    if (!isValidEmail(data.email)) {
      throw new ValidationError('INVALID_EMAIL', 'Email format is invalid');
    }

    strapiBeforeCreateHook(event);
  },

  async afterCreate(event) {
    strapiAfterCreateHook(event);
  },

  async beforeUpdate(event) {
    const { data, params } = event;

    if (params.where && params.where.id) {
      const user = await strapi.entityService.findOne('api::user.user', params.where.id);
      if (!user) {
        throw new NotFoundError('User not found');
      }
    }

    if (data.email && !isValidEmail(data.email)) {
      throw new ValidationError('INVALID_EMAIL', 'Email format is invalid');
    }

    strapiBeforeUpdateHook(event);
  },

  async afterUpdate(event) {
    strapiAfterUpdateHook(event);
  },

  async beforeDelete(event) {
    const token = event.params?.context?.state?.user?.token;
    if (!token || !isValidToken(token)) {
      throw new InvalidTokenError('Invalid or expired token');
    }

    strapiBeforeDeleteHook(event);
  },

  async afterDelete(event) {
    strapiAfterDeleteHook(event);
  },
};
```

## API Reference

### `strapiErrorHandler`

Strapi error handling handler. Catches all errors and maps them to standardized JSON responses.

```typescript
function strapiErrorHandler(error: Error, context: { method: string; path: string; params: EventParams }): any;
```

### `createStrapiErrorHandler(options?: StrapiErrorHandlerOptions)`

Creates a customized error handler.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `logErrors` | `boolean` | `true` | Enable/disable error logging |
| `logStack` | `boolean` | `false` | Include stack traces in logs |
| `includeCause` | `boolean` | `true` | Include error cause in response |
| `includeStack` | `boolean` | `false` | Include stack traces in response (dev only) |
| `defaultStatusCode` | `number` | `500` | Default status for unknown errors |
| `logger` | `function` | `console.error` | Custom logger function |

### `StrapiResponseAdapter`

Static utility class for sending responses.

| Method | HTTP Status | Description |
|--------|-------------|-------------|
| `sendSuccess(response, apiSuccess)` | varies | Send raw ApiSuccess |
| `sendError(response, apiError)` | varies | Send raw ApiError |
| `ok(response, data?, message?)` | 200 | Send success response with data |
| `created(response, data?, message?)` | 201 | Send created response with data |
| `noContent(response, message?)` | 204 | Send no content response |
| `badRequest(response, errorCode, message)` | 400 | Send bad request error |
| `unauthorized(response, errorCode, message)` | 401 | Send unauthorized error |
| `forbidden(response, errorCode, message)` | 403 | Send forbidden error |
| `notFound(response, errorCode, message)` | 404 | Send not found error |
| `conflict(response, errorCode, message)` | 409 | Send conflict error |
| `unprocessableEntity(response, errorCode, message)` | 422 | Send unprocessable entity error |
| `tooManyRequests(response, errorCode, message)` | 429 | Send too many requests error |
| `internalServerError(response, errorCode, message)` | 500 | Send internal server error |
| `badGateway(response, errorCode, message)` | 502 | Send bad gateway error |
| `serviceUnavailable(response, errorCode, message)` | 503 | Send service unavailable error |
| `gatewayTimeout(response, errorCode, message)` | 504 | Send gateway timeout error |

### `withStrapiErrorHandling(middleware, options?)`

Wraps a Strapi lifecycle middleware with error handling.

```typescript
function withStrapiErrorHandling<T extends StrapiLifecycleMiddleware>(
  middleware: T,
  options?: StrapiLifecycleMiddlewareOptions
): StrapiLifecycleMiddleware;
```

### Lifecycle Hooks

| Hook | Description |
|------|-------------|
| `strapiBeforeCreateHook(context)` | Run logic before create |
| `strapiAfterCreateHook(context)` | Run logic after create |
| `strapiBeforeUpdateHook(context)` | Run logic before update |
| `strapiAfterUpdateHook(context)` | Run logic after update |
| `strapiBeforeDeleteHook(context)` | Run logic before delete |
| `strapiAfterDeleteHook(context)` | Run logic after delete |

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

Full TypeScript support with strict mode enabled. The adapter extends Strapi types:

```typescript
import type { Core, EventParams, Module } from '@strapi/strapi';

// Lifecycle context type augmentation
interface StrapiLifecycleContext<T = any> {
  state: Core['state'];
  action: string;
  params: EventParams;
  data: T;
  result?: T;
  error?: Error;
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

| Package Version | Node.js | Strapi | @syncafricabs/kernspark-core |
|----------------|---------|--------|----------------------------------|
| 1.0.0 | >=18.0.0 | ^4.0.0 | ^1.0.0 |

| Feature | Strapi 4+ |
|---------|-------------|
| Lifecycle middleware | Supported |
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
cd kernspark/packages/kernspark-strapi

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


