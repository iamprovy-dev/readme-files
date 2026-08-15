# @syncafricabs/kernspark-feathers

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue)](https://www.typescriptlang.org/)
[![FeathersJS](https://img.shields.io/badge/FeathersJS-5%2B-black)](https://feathersjs.com/)

FeathersJS adapter for the [SyncAfrica KernSpark](https://github.com/iamprovy-dev/kernspark-js) ecosystem. This package provides production-ready FeathersJS service hooks, error handlers, and response utilities that integrate the framework-independent `@syncafricabs/kernspark-core` with FeathersJS.

## Table of Contents

- [What is it?](#what-is-it)
- [Why it exists](#why-it-exists)
- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Comprehensive Usage](#comprehensive-usage)
  - [Service Hooks](#service-hooks)
  - [Error Handling](#error-handling)
  - [Response Adapter](#response-adapter)
  - [Complete Service Example](#complete-service-example)
- [API Reference](#api-reference)
- [Error Handling Reference](#error-handling-reference)
- [TypeScript Support](#typescript-support)
- [Compatibility Matrix](#compatibility-matrix)
- [Contributing](#contributing)
- [License](#license)
- [Author](#author)

## What is it?

`@syncafricabs/kernspark-feathers` is a framework adapter that bridges the [SyncAfrica KernSpark Core](https://github.com/iamprovy-dev/kernspark-js/tree/main/packages/kernspark-core) with FeathersJS. It provides:

- **FeathersIntegration** - Maps `ApplicationError` instances to standardized FeathersJS error responses
- **FeathersResponseAdapter** - Helper methods for sending consistent success and error responses
- **withFeathersErrorHandling** - Higher-order function for wrapping service hooks with error handling
- **feathersValidationHook** - Pre-configured validation hook

The adapter follows the **adapter pattern**: it depends on `@syncafricabs/kernspark-core` (the framework-independent layer) and FeathersJS only in the adapter layer. The core has zero framework dependencies.

## Why it exists

Modern web applications benefit from a **KernSpark** - a common set of contracts, types, and behaviors shared across bounded contexts. The SyncAfrica KernSpark provides:

1. **Standardized API envelopes** (`ApiSuccess` / `ApiError`)
2. **Rich domain error hierarchy** (`ValidationError`, `BusinessError`, `AuthenticationError`, etc.)
3. **Result types** (`Result`, `Ok`, `Err`)
4. **Domain primitives** (`UUID`, `Money`, `Entity`, `ValueObject`)

Without adapters, each team would reimplement framework-specific plumbing to use these core types. This adapter eliminates that duplication by providing FeathersJS-specific bindings.

## Features

- **Service hooks support** - Drop-in FeathersJS service hook error handling
- **Standardized error responses** - All errors follow the `ApiError` envelope
- **Zero-configuration error handling** - Wraps service hooks automatically
- **Full TypeScript support** - Comprehensive type definitions included
- **Production-ready** - Proper error handling, stack traces, cause chains
- **Framework isolation** - Core has no FeathersJS dependencies

## Installation

```bash
npm install @syncafricabs/kernspark-feathers @syncafricabs/kernspark-core @feathersjs/feathers
```

Or with your preferred package manager:

```bash
yarn add @syncafricabs/kernspark-feathers @syncafricabs/kernspark-core @feathersjs/feathers
```

```bash
pnpm add @syncafricabs/kernspark-feathers @syncafricabs/kernspark-core @feathersjs/feathers
```

## Quick Start

```typescript
// src/services/users/users.service.ts
import type { ServiceHook } from '@feathersjs/feathers';
import { withFeathersErrorHandling, FeathersResponseAdapter } from '@syncafricabs/kernspark-feathers';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

export default {
  async find(params) {
    return params.data;
  },

  async get(id, params) {
    const user = await findUser(id);
    if (!user) {
      throw new NotFoundError('User not found');
    }
    return FeathersResponseAdapter.ok(params, user, 'User retrieved');
  },

  async create(data, params) {
    if (!data.name || !data.email) {
      throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
    }
    const user = await createUser(data);
    return FeathersResponseAdapter.created(params, user, 'User created');
  },
};
```

## Comprehensive Usage

### Service Hooks

#### Basic Hook

```typescript
import type { ServiceHook } from '@feathersjs/feathers';
import { withFeathersErrorHandling, FeathersResponseAdapter } from '@syncafricabs/kernspark-feathers';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

const findUser: ServiceHook = withFeathersErrorHandling(async (context) => {
  const user = await User.find(context.params.query);
  context.result = user;
  return context;
});

const getUser: ServiceHook = withFeathersErrorHandling(async (context) => {
  const user = await User.get(context.id, context.params);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  return FeathersResponseAdapter.ok(context, user, 'User retrieved');
});
```

#### Validation Hook

```typescript
import type { ServiceHook } from '@feathersjs/feathers';
import { feathersValidationHook, withFeathersErrorHandling } from '@syncafricabs/kernspark-feathers';
import { ValidationError, ConflictError } from '@syncafricabs/kernspark-core';

const validateUser: ServiceHook = withFeathersErrorHandling(async (context) => {
  if (context.method === 'create' || context.method === 'update') {
    const { name, email } = context.data;

    if (!name || !email) {
      throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
    }

    if (!isValidEmail(email)) {
      throw new ValidationError('INVALID_EMAIL', 'Email format is invalid');
    }
  }

  return context;
});
```

### Error Handling

#### Custom Error Handler Configuration

```typescript
import { createFeathersErrorHandler, FeathersErrorHandlerOptions } from '@syncafricabs/kernspark-feathers';

const options: FeathersErrorHandlerOptions = {
  logErrors: true,
  logStack: true,
  includeCause: true,
  includeStack: process.env.NODE_ENV === 'development',
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
```

### Response Adapter

#### Success Responses

```typescript
import { FeathersResponseAdapter } from '@syncafricabs/kernspark-feathers';

// 200 OK with data
FeathersResponseAdapter.ok(context, user, 'User retrieved');

// 201 Created with data
FeathersResponseAdapter.created(context, user, 'User created');

// 204 No Content
FeathersResponseAdapter.noContent(context);
```

#### Error Responses

```typescript
import { FeathersResponseAdapter } from '@syncafricabs/kernspark-feathers';

FeathersResponseAdapter.badRequest(context, 'MISSING_FIELDS', 'Email and password are required');
FeathersResponseAdapter.unauthorized(context, 'INVALID_CREDENTIALS', 'Invalid email or password');
FeathersResponseAdapter.forbidden(context, 'PERMISSION_DENIED', 'You do not have permission');
FeathersResponseAdapter.notFound(context, 'NOT_FOUND', 'User not found');
FeathersResponseAdapter.conflict(context, 'CONFLICT', 'Resource already exists');
FeathersResponseAdapter.unprocessableEntity(context, 'INVALID_DATA', 'Invalid request data');
FeathersResponseAdapter.tooManyRequests(context, 'RATE_LIMIT_EXCEEDED', 'Too many requests');
FeathersResponseAdapter.internalServerError(context, 'INTERNAL_ERROR', 'An unexpected error occurred');
FeathersResponseAdapter.badGateway(context, 'BAD_GATEWAY', 'Upstream service error');
FeathersResponseAdapter.serviceUnavailable(context, 'SERVICE_UNAVAILABLE', 'Service temporarily unavailable');
FeathersResponseAdapter.gatewayTimeout(context, 'GATEWAY_TIMEOUT', 'Upstream service timeout');
```

### Complete Service Example

```typescript
// src/services/users/users.service.ts
import type { Params, ServiceInterface } from '@feathersjs/feathers';
import { withFeathersErrorHandling, FeathersResponseAdapter } from '@syncafricabs/kernspark-feathers';
import { NotFoundError, ValidationError, ConflictError, InvalidTokenError } from '@syncafricabs/kernspark-core';

class UsersService implements ServiceInterface<any> {
  async find(params: Params) {
    return params.data;
  }

  async get(id: string, params: Params) {
    const user = await User.find(id);
    if (!user) {
      throw new NotFoundError('User not found');
    }
    return FeathersResponseAdapter.ok(params, user, 'User retrieved');
  }

  async create(data: any, params: Params) {
    if (!data.name || !data.email) {
      throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
    }

    if (await User.findBy('email', data.email)) {
      throw new ConflictError('User with this email already exists');
    }

    const user = await User.create(data);
    return FeathersResponseAdapter.created(params, user, 'User created');
  }

  async update(id: string, data: any, params: Params) {
    const user = await User.find(id);
    if (!user) {
      throw new NotFoundError('User not found');
    }

    const updated = await user.update(data);
    return FeathersResponseAdapter.ok(params, updated, 'User updated');
  }

  async patch(id: string, data: any, params: Params) {
    return this.update(id, data, params);
  }

  async remove(id: string, params: Params) {
    const token = params.headers?.authorization;
    if (!token || !isValidToken(token)) {
      throw new InvalidTokenError('Invalid or expired token');
    }

    const user = await User.find(id);
    if (!user) {
      throw new NotFoundError('User not found');
    }

    await user.delete();
    return FeathersResponseAdapter.noContent(params, 'User deleted');
  }
}

export default withFeathersErrorHandling(new UsersService());
```

## API Reference

### `feathersErrorHandler`

FeathersJS error handling handler. Catches all errors and maps them to standardized error responses.

```typescript
function feathersErrorHandler(error: Error, context: HookContext): HookContext;
```

### `createFeathersErrorHandler(options?: FeathersErrorHandlerOptions)`

Creates a customized error handler.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `logErrors` | `boolean` | `true` | Enable/disable error logging |
| `logStack` | `boolean` | `false` | Include stack traces in logs |
| `includeCause` | `boolean` | `true` | Include error cause in response |
| `includeStack` | `boolean` | `false` | Include stack traces in response (dev only) |
| `defaultStatusCode` | `number` | `500` | Default status for unknown errors |
| `logger` | `function` | `console.error` | Custom logger function |

### `FeathersResponseAdapter`

Static utility class for sending responses.

| Method | HTTP Status | Description |
|--------|-------------|-------------|
| `sendSuccess(context, apiSuccess)` | varies | Send raw ApiSuccess |
| `sendError(context, apiError)` | varies | Send raw ApiError |
| `ok(context, data?, message?)` | 200 | Send success response with data |
| `created(context, data?, message?)` | 201 | Send created response with data |
| `noContent(context, message?)` | 204 | Send no content response |
| `badRequest(context, errorCode, message)` | 400 | Send bad request error |
| `unauthorized(context, errorCode, message)` | 401 | Send unauthorized error |
| `forbidden(context, errorCode, message)` | 403 | Send forbidden error |
| `notFound(context, errorCode, message)` | 404 | Send not found error |
| `conflict(context, errorCode, message)` | 409 | Send conflict error |
| `unprocessableEntity(context, errorCode, message)` | 422 | Send unprocessable entity error |
| `tooManyRequests(context, errorCode, message)` | 429 | Send too many requests error |
| `internalServerError(context, errorCode, message)` | 500 | Send internal server error |
| `badGateway(context, errorCode, message)` | 502 | Send bad gateway error |
| `serviceUnavailable(context, errorCode, message)` | 503 | Send service unavailable error |
| `gatewayTimeout(context, errorCode, message)` | 504 | Send gateway timeout error |

### `withFeathersErrorHandling(hook, options?)`

Wraps a Feathers service hook with error handling.

```typescript
function withFeathersErrorHandling<T extends FeathersServiceHook>(
  hook: T,
  options?: FeathersServiceHookOptions
): FeathersServiceHook;
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

Full TypeScript support with strict mode enabled. The adapter extends FeathersJS types:

```typescript
import type { HookContext, HookResult, ServiceInterface, Params } from '@feathersjs/feathers';

// Service hook type augmentation
interface ServiceHook<T = any> {
  (context: HookContext<T>): HookResult | Promise<HookResult>;
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

| Package Version | Node.js | FeathersJS | @syncafricabs/kernspark-core |
|----------------|---------|------------|----------------------------------|
| 1.0.0 | >=18.0.0 | ^5.0.0 | ^1.0.0 |

| Feature | FeathersJS 5+ |
|---------|---------------|
| Service hooks | Supported |
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
cd kernspark/packages/kernspark-feathers

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


