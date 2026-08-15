# @syncafricabs/kernspark-remix

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue)](https://www.typescriptlang.org/)
[![Remix](https://img.shields.io/badge/Remix-2%2B-1212E0)](https://remix.run/)

Remix adapter for the [SyncAfrica KernSpark](https://github.com/iamprovy-dev/kernspark-js) ecosystem. This package provides production-ready Remix loader/action wrappers, error handlers, and response utilities that integrate the framework-independent `@syncafricabs/kernspark-core` with Remix.

## Table of Contents

- [What is it?](#what-is-it)
- [Why it exists](#why-it-exists)
- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Comprehensive Usage](#comprehensive-usage)
  - [Loaders](#loaders)
  - [Actions](#actions)
  - [Error Handling](#error-handling)
  - [Response Adapter](#response-adapter)
  - [Error Boundaries](#error-boundaries)
- [API Reference](#api-reference)
- [Error Handling Reference](#error-handling-reference)
- [TypeScript Support](#typescript-support)
- [Compatibility Matrix](#compatibility-matrix)
- [Contributing](#contributing)
- [License](#license)
- [Author](#author)

## What is it?

`@syncafricabs/kernspark-remix` is a framework adapter that bridges the [SyncAfrica KernSpark Core](https://github.com/iamprovy-dev/kernspark-js/tree/main/packages/kernspark-core) with Remix. It provides:

- **RemixIntegration** - Maps `ApplicationError` instances to standardized JSON API responses
- **RemixResponseAdapter** - Helper methods for sending consistent success and error responses
- **withErrorHandling** - Higher-order function for wrapping loaders with error handling
- **withActionErrorHandling** - Higher-order function for wrapping actions with error handling

The adapter follows the **adapter pattern**: it depends on `@syncafricabs/kernspark-core` (the framework-independent layer) and Remix only in the adapter layer. The core has zero framework dependencies.

## Why it exists

Modern web applications benefit from a **KernSpark** - a common set of contracts, types, and behaviors shared across bounded contexts. The SyncAfrica KernSpark provides:

1. **Standardized API envelopes** (`ApiSuccess` / `ApiError`)
2. **Rich domain error hierarchy** (`ValidationError`, `BusinessError`, `AuthenticationError`, etc.)
3. **Result types** (`Result`, `Ok`, `Err`)
4. **Domain primitives** (`UUID`, `Money`, `Entity`, `ValueObject`)

Without adapters, each team would reimplement framework-specific plumbing to use these core types. This adapter eliminates that duplication by providing Remix-specific bindings.

## Features

- **Loader support** - Drop-in error handling for Remix loaders
- **Action support** - Drop-in error handling for Remix actions
- **Standardized JSON responses** - All responses follow the `ApiSuccess` / `ApiError` envelope
- **Zero-configuration error handling** - Wraps loaders and actions automatically
- **Full TypeScript support** - Comprehensive type definitions included
- **Production-ready** - Proper error handling, stack traces, cause chains
- **Framework isolation** - Core has no Remix dependencies

## Installation

```bash
npm install @syncafricabs/kernspark-remix @syncafricabs/kernspark-core @remix-run/node
```

Or with your preferred package manager:

```bash
yarn add @syncafricabs/kernspark-remix @syncafricabs/kernspark-core @remix-run/node
```

```bash
pnpm add @syncafricabs/kernspark-remix @syncafricabs/kernspark-core @remix-run/node
```

## Quick Start

```typescript
// app/routes/users.$id.tsx
import { json } from '@remix-run/node';
import { withErrorHandling, RemixResponseAdapter } from '@syncafricabs/kernspark-remix';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

export const loader = withErrorHandling(async ({ params }) => {
  const user = findUser(params.id);

  if (!user) {
    throw new NotFoundError('User not found');
  }

  return RemixResponseAdapter.sendSuccess(user, 'User retrieved successfully');
});

export const action = withActionErrorHandling(async ({ request }) => {
  const body = await request.json();

  if (!body.name || !body.email) {
    throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
  }

  const user = createUser(body);
  return RemixResponseAdapter.created(user, 'User created successfully');
});
```

## Comprehensive Usage

### Loaders

#### Basic Loader with Error Handling

```typescript
// app/routes/users.$id.tsx
import { json } from '@remix-run/node';
import { withErrorHandling, RemixResponseAdapter } from '@syncafricabs/kernspark-remix';
import { NotFoundError } from '@syncafricabs/kernspark-core';

export const loader = withErrorHandling(async ({ params }) => {
  const user = findUser(params.id);

  if (!user) {
    throw new NotFoundError('User not found');
  }

  return RemixResponseAdapter.sendSuccess(user, 'User retrieved successfully');
});
```

#### Loader with Query Parameters

```typescript
// app/routes/users.tsx
import { withErrorHandling, RemixResponseAdapter } from '@syncafricabs/kernspark-remix';
import { ValidationError, BusinessError } from '@syncafricabs/kernspark-core';

export const loader = withErrorHandling(async ({ request }) => {
  const url = new URL(request.url);
  const page = parseInt(url.searchParams.get('page') || '1');
  const limit = parseInt(url.searchParams.get('limit') || '10');

  if (isNaN(page) || isNaN(limit)) {
    throw new ValidationError('INVALID_PAGINATION', 'Page and limit must be numbers');
  }

  if (limit > 100) {
    throw new BusinessError('PAGE_SIZE_EXCEEDED', 400, 'Maximum page size is 100');
  }

  const users = await getUsers(page, limit);
  return RemixResponseAdapter.sendSuccess(users, 'Users retrieved successfully');
});
```

### Actions

#### POST Action with Validation

```typescript
// app/routes/users.tsx
import { withActionErrorHandling, RemixResponseAdapter } from '@syncafricabs/kernspark-remix';
import { ValidationError, ConflictError, BusinessError } from '@syncafricabs/kernspark-core';

export const action = withActionErrorHandling(async ({ request }) => {
  const body = await request.json();

  if (!body.name || !body.email) {
    throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
  }

  if (!isValidEmail(body.email)) {
    throw new ValidationError('INVALID_EMAIL', 'Email format is invalid');
  }

  const existingUser = await findUserByEmail(body.email);
  if (existingUser) {
    throw new ConflictError('User with this email already exists');
  }

  const user = await createUser(body);
  return RemixResponseAdapter.created(user, 'User created successfully');
});
```

#### DELETE Action with Authorization

```typescript
// app/routes/users.$id.tsx
import { withActionErrorHandling, RemixResponseAdapter } from '@syncafricabs/kernspark-remix';
import { InvalidTokenError, PermissionDeniedError, NotFoundError } from '@syncafricabs/kernspark-core';

export const action = withActionErrorHandling(async ({ request, params }) => {
  const token = request.headers.get('authorization');
  if (!token || !isValidToken(token)) {
    throw new InvalidTokenError('Invalid or expired token');
  }

  const user = await findUser(params.id);
  if (!user) {
    throw new NotFoundError('User not found');
  }

  if (!canDeleteUser(token, user.id)) {
    throw new PermissionDeniedError('You do not have permission to delete users');
  }

  await deleteUser(user.id);
  return RemixResponseAdapter.noContent('User deleted successfully');
});
```

### Error Handling

#### Custom Error Handler Configuration

```typescript
import { createRemixErrorHandler, RemixErrorHandlerOptions } from '@syncafricabs/kernspark-remix';

const options: RemixErrorHandlerOptions = {
  logErrors: true,
  logStack: true,
  includeCause: true,
  includeStack: process.env.NODE_ENV === 'development',
  defaultStatusCode: 500,
  logger: (error, context) => {
    console.error({
      url: new URL(context.request.url).pathname,
      method: context.request.method,
      params: context.params,
      error: error.message,
      stack: error.stack,
    });
  },
};

export const loader = withErrorHandling(myLoader, options);
```

### Response Adapter

#### Success Responses

```typescript
import { RemixResponseAdapter } from '@syncafricabs/kernspark-remix';

// 200 OK with data
return RemixResponseAdapter.sendSuccess(user, 'User retrieved');

// 201 Created with data
return RemixResponseAdapter.created(user, 'User created');

// 204 No Content
return RemixResponseAdapter.noContent();
```

#### Error Responses

```typescript
import { RemixResponseAdapter } from '@syncafricabs/kernspark-remix';

return RemixResponseAdapter.badRequest('MISSING_FIELDS', 'Email and password are required');
return RemixResponseAdapter.unauthorized('INVALID_CREDENTIALS', 'Invalid email or password');
return RemixResponseAdapter.forbidden('PERMISSION_DENIED', 'You do not have permission');
return RemixResponseAdapter.notFound('NOT_FOUND', 'User not found');
return RemixResponseAdapter.conflict('CONFLICT', 'Resource already exists');
return RemixResponseAdapter.unprocessableEntity('INVALID_DATA', 'Invalid request data');
return RemixResponseAdapter.tooManyRequests('RATE_LIMIT_EXCEEDED', 'Too many requests');
return RemixResponseAdapter.internalServerError('INTERNAL_ERROR', 'An unexpected error occurred');
return RemixResponseAdapter.badGateway('BAD_GATEWAY', 'Upstream service error');
return RemixResponseAdapter.serviceUnavailable('SERVICE_UNAVAILABLE', 'Service temporarily unavailable');
return RemixResponseAdapter.gatewayTimeout('GATEWAY_TIMEOUT', 'Upstream service timeout');
```

### Error Boundaries

```typescript
// app/root.tsx
import { LoaderFunctionArgs, ActionFunctionArgs } from '@remix-run/node';
import {
  Links,
  LiveReload,
  Meta,
  Outlet,
  Scripts,
  ScrollRestoration,
} from '@remix-run/react';
import { withErrorHandling, withActionErrorHandling } from '@syncafricabs/kernspark-remix';

export const loader = withErrorHandling(async ({ request }) => {
  return json({ date: new Date() });
});

export const action = withActionErrorHandling(async ({ request }) => {
  return json({ success: true });
});

export default function App() {
  return (
    <html lang="en">
      <head>
        <Meta />
        <Links />
      </head>
      <body>
        <Outlet />
        <ScrollRestoration />
        <Scripts />
        <LiveReload />
      </body>
    </html>
  );
}
```

## API Reference

### `remixErrorHandler`

Remix error handling handler. Catches all errors and maps them to standardized JSON responses.

```typescript
function remixErrorHandler(error: Error, context: { request: Request; params: Record<string, string | undefined> }): Response;
```

### `createRemixErrorHandler(options?: RemixErrorHandlerOptions)`

Creates a customized error handler.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `logErrors` | `boolean` | `true` | Enable/disable error logging |
| `logStack` | `boolean` | `false` | Include stack traces in logs |
| `includeCause` | `boolean` | `true` | Include error cause in response |
| `includeStack` | `boolean` | `false` | Include stack traces in response (dev only) |
| `defaultStatusCode` | `number` | `500` | Default status for unknown errors |
| `logger` | `function` | `console.error` | Custom logger function |

### `RemixResponseAdapter`

Static utility class for sending responses.

| Method | HTTP Status | Description |
|--------|-------------|-------------|
| `sendSuccess(data, message?, statusCode?)` | varies | Send success response with data |
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

### `withErrorHandling(loader)`

Wraps a Remix loader with error handling.

```typescript
function withErrorHandling<T extends RemixLoader>(loader: T): RemixLoader;
```

### `withActionErrorHandling(action)`

Wraps a Remix action with error handling.

```typescript
function withActionErrorHandling<T extends RemixAction>(action: T): RemixAction;
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

Full TypeScript support with strict mode enabled. The adapter extends Remix types:

```typescript
import { LoaderFunctionArgs, ActionFunctionArgs, json } from '@remix-run/node';

// Loader type augmentation
interface AppLoaderData {
  user: User;
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

| Package Version | Node.js | Remix | @syncafricabs/kernspark-core |
|----------------|---------|-------|----------------------------------|
| 1.0.0 | >=18.0.0 | ^2.0.0 | ^1.0.0 |

| Feature | Remix 2+ |
|---------|-------------|
| Loader error handling | Supported |
| Action error handling | Supported |
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
cd kernspark/packages/kernspark-remix

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


