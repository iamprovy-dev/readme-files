# @syncafricabs/kernspark-sveltekit

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue)](https://www.typescriptlang.org/)
[![SvelteKit](https://img.shields.io/badge/SvelteKit-1%2B2%2B-FF3E00)](https://kit.svelte.dev/)

SvelteKit adapter for the [SyncAfrica KernSpark](https://github.com/iamprovy-dev/kernspark-js) ecosystem. This package provides production-ready SvelteKit `handle` functions, error handlers, and response utilities that integrate the framework-independent `@syncafricabs/kernspark-core` with SvelteKit.

## Table of Contents

- [What is it?](#what-is-it)
- [Why it exists](#why-it-exists)
- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Comprehensive Usage](#comprehensive-usage)
  - [Handle Function](#handle-function)
  - [Error Handling](#error-handling)
  - [Response Adapter](#response-adapter)
  - [Form Actions](#form-actions)
- [API Reference](#api-reference)
- [Error Handling Reference](#error-handling-reference)
- [TypeScript Support](#typescript-support)
- [Compatibility Matrix](#compatibility-matrix)
- [Contributing](#contributing)
- [License](#license)
- [Author](#author)

## What is it?

`@syncafricabs/kernspark-sveltekit` is a framework adapter that bridges the [SyncAfrica KernSpark Core](https://github.com/iamprovy-dev/kernspark-js/tree/main/packages/kernspark-core) with SvelteKit. It provides:

- **SvelteKitIntegration** - Maps `ApplicationError` instances to standardized JSON API responses
- **SvelteKitResponseAdapter** - Helper methods for sending consistent success and error responses
- **svelteKitHandle** - SvelteKit `handle` function with integrated error handling
- **createSvelteKitHandle** - Configurable SvelteKit handle factory

The adapter follows the **adapter pattern**: it depends on `@syncafricabs/kernspark-core` (the framework-independent layer) and SvelteKit only in the adapter layer. The core has zero framework dependencies.

## Why it exists

Modern web applications benefit from a **KernSpark** - a common set of contracts, types, and behaviors shared across bounded contexts. The SyncAfrica KernSpark provides:

1. **Standardized API envelopes** (`ApiSuccess` / `ApiError`)
2. **Rich domain error hierarchy** (`ValidationError`, `BusinessError`, `AuthenticationError`, etc.)
3. **Result types** (`Result`, `Ok`, `Err`)
4. **Domain primitives** (`UUID`, `Money`, `Entity`, `ValueObject`)

Without adapters, each team would reimplement framework-specific plumbing to use these core types. This adapter eliminates that duplication by providing SvelteKit-specific bindings.

## Features

- **Handle function support** - Integrated error handling in SvelteKit's universal handle
- **Standardized JSON responses** - All responses follow the `ApiSuccess` / `ApiError` envelope
- **Zero-configuration error handling** - Drop-in handle function
- **Full TypeScript support** - Comprehensive type definitions included
- **Production-ready** - Proper error handling, stack traces, cause chains
- **Framework isolation** - Core has no SvelteKit dependencies

## Installation

```bash
npm install @syncafricabs/kernspark-sveltekit @syncafricabs/kernspark-core @sveltejs/kit
```

Or with your preferred package manager:

```bash
yarn add @syncafricabs/kernspark-sveltekit @syncafricabs/kernspark-core @sveltejs/kit
```

```bash
pnpm add @syncafricabs/kernspark-sveltekit @syncafricabs/kernspark-core @sveltejs/kit
```

## Quick Start

```typescript
// src/hooks.ts
import { svelteKitHandle } from '@syncafricabs/kernspark-sveltekit';

export const handle = svelteKitHandle;
```

```typescript
// src/routes/api/users/+server.ts
import { SvelteKitResponseAdapter } from '@syncafricabs/kernspark-sveltekit';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

export async function GET({ url }) {
  const id = url.searchParams.get('id');
  const user = findUser(id);

  if (!user) {
    throw new NotFoundError('User not found');
  }

  return SvelteKitResponseAdapter.sendSuccess(user, 'User retrieved successfully');
}

export async function POST({ request }) {
  const body = await request.json();

  if (!body.name || !body.email) {
    throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
  }

  const user = createUser(body);
  return SvelteKitResponseAdapter.created(user, 'User created successfully');
}
```

## Comprehensive Usage

### Handle Function

#### Basic Setup

```typescript
// src/hooks.ts
import { svelteKitHandle } from '@syncafricabs/kernspark-sveltekit';

export const handle = svelteKitHandle;
```

#### Custom Handle Configuration

```typescript
// src/hooks.ts
import { createSvelteKitHandle } from '@syncafricabs/kernspark-sveltekit';

export const handle = createSvelteKitHandle({
  errorHandling: true,
});
```

#### Handle with Event Data

```typescript
// src/hooks.ts
import { createSvelteKitHandle } from '@syncafricabs/kernspark-sveltekit';

export const handle = createSvelteKitHandle({
  errorHandling: true,
});

// Access event data in load functions
export const load = async ({ fetch, url }) => {
  const correlationId = url.searchParams.get('correlationId');
  // ...
};
```

### Error Handling

#### Throwing Errors from Load Functions

```typescript
// src/routes/api/users/[id]/+page.ts
import { SvelteKitResponseAdapter } from '@syncafricabs/kernspark-sveltekit';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

export const load = async ({ params, fetch }) => {
  const user = await fetch(`/api/users/${params.id}`).then(r => r.json());

  if (!user) {
    throw new NotFoundError('User not found');
  }

  return {
    user,
    message: 'User loaded successfully'
  };
};
```

#### Throwing Errors from Form Actions

```typescript
// src/routes/login/+page.ts
import { SvelteKitResponseAdapter } from '@syncafricabs/kernspark-sveltekit';
import { ValidationError, InvalidTokenError, PermissionDeniedError } from '@syncafricabs/kernspark-core';

export const actions = {
  default: async ({ request }) => {
    const data = await request.formData();
    const email = data.get('email') as string;
    const password = data.get('password') as string;

    if (!email || !password) {
      throw new ValidationError('MISSING_FIELDS', 'Email and password are required');
    }

    const token = await authenticate(email, password);
    if (!token) {
      throw new InvalidTokenError('Invalid email or password');
    }

    return SvelteKitResponseAdapter.ok({ token }, 'Login successful');
  },
};
```

### Response Adapter

#### Success Responses

```typescript
import { SvelteKitResponseAdapter } from '@syncafricabs/kernspark-sveltekit';

// 200 OK with data
return SvelteKitResponseAdapter.sendSuccess(user, 'User retrieved');

// 201 Created with data
return SvelteKitResponseAdapter.created(user, 'User created');

// 204 No Content
return SvelteKitResponseAdapter.noContent();
```

#### Error Responses

```typescript
import { SvelteKitResponseAdapter } from '@syncafricabs/kernspark-sveltekit';

return SvelteKitResponseAdapter.badRequest('MISSING_FIELDS', 'Email and password are required');
return SvelteKitResponseAdapter.unauthorized('INVALID_CREDENTIALS', 'Invalid email or password');
return SvelteKitResponseAdapter.forbidden('PERMISSION_DENIED', 'You do not have permission');
return SvelteKitResponseAdapter.notFound('NOT_FOUND', 'User not found');
return SvelteKitResponseAdapter.conflict('CONFLICT', 'Resource already exists');
return SvelteKitResponseAdapter.unprocessableEntity('INVALID_DATA', 'Invalid request data');
return SvelteKitResponseAdapter.tooManyRequests('RATE_LIMIT_EXCEEDED', 'Too many requests');
return SvelteKitResponseAdapter.internalServerError('INTERNAL_ERROR', 'An unexpected error occurred');
return SvelteKitResponseAdapter.badGateway('BAD_GATEWAY', 'Upstream service error');
return SvelteKitResponseAdapter.serviceUnavailable('SERVICE_UNAVAILABLE', 'Service temporarily unavailable');
return SvelteKitResponseAdapter.gatewayTimeout('GATEWAY_TIMEOUT', 'Upstream service timeout');
```

### Form Actions

#### Complete Form Example

```typescript
// src/routes/users/new/+page.ts
import { SvelteKitResponseAdapter } from '@syncafricabs/kernspark-sveltekit';
import { ValidationError, ConflictError, NotFoundError } from '@syncafricabs/kernspark-core';

export const actions = {
  default: async ({ request, fetch }) => {
    const data = await request.formData();
    const name = data.get('name') as string;
    const email = data.get('email') as string;

    if (!name || !email) {
      throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
    }

    const existingUser = await fetch(`/api/users?email=${email}`).then(r => r.json());
    if (existingUser) {
      throw new ConflictError('User with this email already exists');
    }

    const response = await fetch('/api/users', {
      method: 'POST',
      body: JSON.stringify({ name, email }),
      headers: { 'Content-Type': 'application/json' },
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.message);
    }

    const user = await response.json();
    return SvelteKitResponseAdapter.created(user, 'User created successfully');
  },
};
```

## API Reference

### `svelteKitErrorHandler`

SvelteKit error handling handler. Catches all errors and maps them to standardized JSON responses.

```typescript
function svelteKitErrorHandler(error: Error, event: any): Response;
```

### `createSvelteKitErrorHandler(options?: SvelteKitErrorHandlerOptions)`

Creates a customized error handler.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `logErrors` | `boolean` | `true` | Enable/disable error logging |
| `logStack` | `boolean` | `false` | Include stack traces in logs |
| `includeCause` | `boolean` | `true` | Include error cause in response |
| `includeStack` | `boolean` | `false` | Include stack traces in response (dev only) |
| `defaultStatusCode` | `number` | `500` | Default status for unknown errors |
| `logger` | `function` | `console.error` | Custom logger function |

### `SvelteKitResponseAdapter`

Static utility class for sending responses.

| Method | HTTP Status | Description |
|--------|-------------|-------------|
| `sendSuccess(data, message?)` | 200 | Send success response with data |
| `created(data, message?)` | 201 | Send created response with data |
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

### `createSvelteKitHandle(options?: SvelteKitHandleOptions)`

Creates a SvelteKit handle function with error handling.

```typescript
function createSvelteKitHandle(options?: SvelteKitHandleOptions): Handle;
```

### `svelteKitHandle`

Pre-configured SvelteKit handle with error handling enabled.

```typescript
const handle: Handle = svelteKitHandle;
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

Full TypeScript support with strict mode enabled. The adapter extends SvelteKit types:

```typescript
import type { Handle, HandleFetch, ResolveOptions } from '@sveltejs/kit';

// Custom handle type augmentation
declare module '@syncafricabs/kernspark-sveltekit' {
  export interface SvelteKitHandleOptions {
    errorHandling?: boolean;
    logErrors?: boolean;
    includeCause?: boolean;
    includeStack?: boolean;
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

| Package Version | Node.js | SvelteKit | @syncafricabs/kernspark-core |
|----------------|---------|-----------|----------------------------------|
| 1.0.0 | >=18.0.0 | ^1.0.0 | ^1.0.0 |

| Feature | SvelteKit 1+ / 2+ |
|---------|-------------------|
| Handle function | Supported |
| Error handling | Supported |
| Form actions | Supported |
| Load functions | Supported |
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
cd kernspark/packages/kernspark-sveltekit

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


