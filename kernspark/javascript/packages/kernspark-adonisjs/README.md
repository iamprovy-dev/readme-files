# @syncafricabs/kernspark-adonisjs

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue)](https://www.typescriptlang.org/)
[![AdonisJS](https://img.shields.io/badge/AdonisJS-5%2B-5A2D8A)](https://docs.adonisjs.com/)

AdonisJS adapter for the [SyncAfrica KernSpark](https://github.com/iamprovy-dev/kernspark-js) ecosystem. This package provides production-ready AdonisJS exception handlers, response macros, and utilities that integrate the framework-independent `@syncafricabs/kernspark-core` with AdonisJS.

## Table of Contents

- [What is it?](#what-is-it)
- [Why it exists](#why-it-exists)
- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Comprehensive Usage](#comprehensive-usage)
  - [Exception Handler](#exception-handler)
  - [Response Adapter](#response-adapter)
  - [Controllers](#controllers)
  - [Validation](#validation)
- [API Reference](#api-reference)
- [Error Handling Reference](#error-handling-reference)
- [TypeScript Support](#typescript-support)
- [Compatibility Matrix](#compatibility-matrix)
- [Contributing](#contributing)
- [License](#license)
- [Author](#author)

## What is it?

`@syncafricabs/kernspark-adonisjs` is a framework adapter that bridges the [SyncAfrica KernSpark Core](https://github.com/iamprovy-dev/kernspark-js/tree/main/packages/kernspark-core) with AdonisJS. It provides:

- **AdonisJsIntegration** - Maps `ApplicationError` instances to standardized JSON API responses
- **AdonisJsResponseAdapter** - Helper methods for sending consistent success and error responses
- **adonisJsExceptionHandler** - Pre-configured exception handler for AdonisJS
- **withAdonisErrorHandling** - Higher-order function for wrapping controllers with error handling

The adapter follows the **adapter pattern**: it depends on `@syncafricabs/kernspark-core` (the framework-independent layer) and AdonisJS only in the adapter layer. The core has zero framework dependencies.

## Why it exists

Modern web applications benefit from a **KernSpark** - a common set of contracts, types, and behaviors shared across bounded contexts. The SyncAfrica KernSpark provides:

1. **Standardized API envelopes** (`ApiSuccess` / `ApiError`)
2. **Rich domain error hierarchy** (`ValidationError`, `BusinessError`, `AuthenticationError`, etc.)
3. **Result types** (`Result`, `Ok`, `Err`)
4. **Domain primitives** (`UUID`, `Money`, `Entity`, `ValueObject`)

Without adapters, each team would reimplement framework-specific plumbing to use these core types. This adapter eliminates that duplication by providing AdonisJS-specific bindings.

## Features

- **Exception handler support** - Drop-in AdonisJS exception handler
- **Standardized JSON responses** - All responses follow the `ApiSuccess` / `ApiError` envelope
- **Zero-configuration error handling** - Pre-configured exception handler
- **Full TypeScript support** - Comprehensive type definitions included
- **Production-ready** - Proper error handling, stack traces, cause chains
- **Framework isolation** - Core has no AdonisJS dependencies

## Installation

```bash
npm install @syncafricabs/kernspark-adonisjs @syncafricabs/kernspark-core @adonisjs/core
```

Or with your preferred package manager:

```bash
yarn add @syncafricabs/kernspark-adonisjs @syncafricabs/kernspark-core @adonisjs/core
```

```bash
pnpm add @syncafricabs/kernspark-adonisjs @syncafricabs/kernspark-core @adonisjs/core
```

## Quick Start

```typescript
// start/kernel.ts
import { adonisJsExceptionHandler } from '@syncafricabs/kernspark-adonisjs';

export const exceptionHandler = adonisJsExceptionHandler;
```

```typescript
// app/controllers/users_controller.ts
import type { HttpContext } from '@adonisjs/core/http/types';
import { withAdonisErrorHandling, AdonisJsResponseAdapter } from '@syncafricabs/kernspark-adonisjs';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

export default class UsersController {
  async index({ response }: HttpContext) {
    const users = await User.all();
    return AdonisJsResponseAdapter.ok({ response }, users, 'Users retrieved successfully');
  }

  async show({ params, response }: HttpContext) {
    const user = await User.findOrFail(params.id);
    if (!user) {
      throw new NotFoundError('User not found');
    }
    return AdonisJsResponseAdapter.ok({ response }, user, 'User retrieved successfully');
  }

  async store({ request, response }: HttpContext) {
    const data = request.only(['name', 'email']);

    if (!data.name || !data.email) {
      throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
    }

    const user = await User.create(data);
    return AdonisJsResponseAdapter.created({ response }, user, 'User created successfully');
  }
}
```

## Comprehensive Usage

### Exception Handler

#### Basic Setup

```typescript
// start/kernel.ts
import { adonisJsExceptionHandler } from '@syncafricabs/kernspark-adonisjs';

export const exceptionHandler = adonisJsExceptionHandler;
```

#### Custom Exception Handler Configuration

```typescript
// start/kernel.ts
import { createAdonisJsExceptionHandler, AdonisJsExceptionHandlerOptions } from '@syncafricabs/kernspark-adonisjs';

const options: AdonisJsExceptionHandlerOptions = {
  logErrors: true,
  logStack: true,
  includeCause: true,
  includeStack: process.env.NODE_ENV === 'development',
  defaultStatusCode: 500,
  logger: (error, ctx) => {
    console.error({
      method: ctx.request.method(),
      url: ctx.request.url().pathname,
      error: error.message,
      stack: error.stack,
    });
  },
};

export const exceptionHandler = createAdonisJsExceptionHandler(options);
```

### Response Adapter

#### Success Responses

```typescript
import { AdonisJsResponseAdapter } from '@syncafricabs/kernspark-adonisjs';

// 200 OK with data
AdonisJsResponseAdapter.ok({ response }, user, 'User retrieved');

// 201 Created with data
AdonisJsResponseAdapter.created({ response }, user, 'User created');

// 204 No Content
AdonisJsResponseAdapter.noContent({ response });
```

#### Error Responses

```typescript
import { AdonisJsResponseAdapter } from '@syncafricabs/kernspark-adonisjs';

AdonisJsResponseAdapter.badRequest({ response }, 'MISSING_FIELDS', 'Email and password are required');
AdonisJsResponseAdapter.unauthorized({ response }, 'INVALID_CREDENTIALS', 'Invalid email or password');
AdonisJsResponseAdapter.forbidden({ response }, 'PERMISSION_DENIED', 'You do not have permission');
AdonisJsResponseAdapter.notFound({ response }, 'NOT_FOUND', 'User not found');
AdonisJsResponseAdapter.conflict({ response }, 'CONFLICT', 'Resource already exists');
AdonisJsResponseAdapter.unprocessableEntity({ response }, 'INVALID_DATA', 'Invalid request data');
AdonisJsResponseAdapter.tooManyRequests({ response }, 'RATE_LIMIT_EXCEEDED', 'Too many requests');
AdonisJsResponseAdapter.internalServerError({ response }, 'INTERNAL_ERROR', 'An unexpected error occurred');
AdonisJsResponseAdapter.badGateway({ response }, 'BAD_GATEWAY', 'Upstream service error');
AdonisJsResponseAdapter.serviceUnavailable({ response }, 'SERVICE_UNAVAILABLE', 'Service temporarily unavailable');
AdonisJsResponseAdapter.gatewayTimeout({ response }, 'GATEWAY_TIMEOUT', 'Upstream service timeout');
```

### Controllers

#### Basic Controller

```typescript
// app/controllers/users_controller.ts
import type { HttpContext } from '@adonisjs/core/http/types';
import { withAdonisErrorHandling, AdonisJsResponseAdapter } from '@syncafricabs/kernspark-adonisjs';
import { NotFoundError, ValidationError, ConflictError } from '@syncafricabs/kernspark-core';

export default class UsersController {
  async index({ response }: HttpContext) {
    const users = await User.all();
    return AdonisJsResponseAdapter.ok({ response }, users, 'Users retrieved successfully');
  }

  async show({ params, response }: HttpContext) {
    const user = await User.findOrFail(params.id);
    if (!user) {
      throw new NotFoundError('User not found');
    }
    return AdonisJsResponseAdapter.ok({ response }, user, 'User retrieved successfully');
  }

  async store({ request, response }: HttpContext) {
    const data = request.only(['name', 'email']);

    if (!data.name || !data.email) {
      throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
    }

    if (await User.findBy('email', data.email)) {
      throw new ConflictError('User with this email already exists');
    }

    const user = await User.create(data);
    return AdonisJsResponseAdapter.created({ response }, user, 'User created successfully');
  }
}
```

#### Controller with Error Handling Wrapper

```typescript
// app/controllers/users_controller.ts
import type { HttpContext } from '@adonisjs/core/http/types';
import { withAdonisErrorHandling, AdonisJsResponseAdapter } from '@syncafricabs/kernspark-adonisjs';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

class UsersController {
  async show({ params }: HttpContext) {
    const user = await User.findOrFail(params.id);
    if (!user) {
      throw new NotFoundError('User not found');
    }
    return user;
  }

  async store({ request }: HttpContext) {
    const data = request.only(['name', 'email']);

    if (!data.name || !data.email) {
      throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
    }

    return await User.create(data);
  }
}

export default withAdonisErrorHandling(new UsersController());
```

### Validation

```typescript
// app/controllers/users_controller.ts
import { ValidationError } from '@syncafricabs/kernspark-core';

export default class UsersController {
  async store({ request, response }: HttpContext) {
    const { name, email, age } = request.only(['name', 'email', 'age']);

    if (!name || name.length < 2) {
      throw new ValidationError('INVALID_NAME', 'Name must be at least 2 characters');
    }

    if (!email || !email.includes('@')) {
      throw new ValidationError('INVALID_EMAIL', 'Email format is invalid');
    }

    if (!age || age < 18) {
      throw new ValidationError('INVALID_AGE', 'You must be at least 18 years old');
    }

    const user = await User.create({ name, email, age });
    return AdonisJsResponseAdapter.created({ response }, user, 'User created successfully');
  }
}
```

## API Reference

### `adonisJsExceptionHandler`

AdonisJS exception handler. Catches all errors and maps them to standardized JSON responses.

```typescript
function adonisJsExceptionHandler(error: Error, ctx: ContextContract): any;
```

### `createAdonisJsExceptionHandler(options?: AdonisJsExceptionHandlerOptions)`

Creates a customized exception handler.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `logErrors` | `boolean` | `true` | Enable/disable error logging |
| `logStack` | `boolean` | `false` | Include stack traces in logs |
| `includeCause` | `boolean` | `true` | Include error cause in response |
| `includeStack` | `boolean` | `false` | Include stack traces in response (dev only) |
| `defaultStatusCode` | `number` | `500` | Default status for unknown errors |
| `logger` | `function` | `console.error` | Custom logger function |

### `AdonisJsResponseAdapter`

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

### `withAdonisErrorHandling(controller)`

Wraps a controller with error handling.

```typescript
function withAdonisErrorHandling<T extends AdonisJsController>(
  controller: T
): AdonisJsController;
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

Full TypeScript support with strict mode enabled. The adapter extends AdonisJS types:

```typescript
import type { HttpContext, ContextContract } from '@adonisjs/core/http/types';

// Context contract type augmentation
interface ContextContract {
  request: {
    method(): string;
    url(): { pathname: string };
    input(): any;
  };
  response: {
    status(code: number): any;
    json(data: any): any;
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

| Package Version | Node.js | AdonisJS | @syncafricabs/kernspark-core |
|----------------|---------|----------|----------------------------------|
| 1.0.0 | >=18.0.0 | ^5.0.0 | ^1.0.0 |

| Feature | AdonisJS 5+ |
|---------|-------------|
| Exception handler | Supported |
| Response macros | Supported |
| Controllers | Supported |
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
cd kernspark/packages/kernspark-adonisjs

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


