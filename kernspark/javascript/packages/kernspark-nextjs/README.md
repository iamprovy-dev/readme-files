# @syncafricabs/kernspark-nextjs

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue)](https://www.typescriptlang.org/)
[![Next.js](https://img.shields.io/badge/Next.js-13%2B-black)](https://nextjs.org/)

Next.js adapter for the [SyncAfrica KernSpark](https://github.com/iamprovy-dev/kernspark-js) ecosystem. This package provides production-ready Next.js API route handlers, error handlers, and response utilities that integrate the framework-independent `@syncafricabs/kernspark-core` with Next.js App Router and Pages Router.

## Table of Contents

- [What is it?](#what-is-it)
- [Why it exists](#why-it-exists)
- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Comprehensive Usage](#comprehensive-usage)
  - [Pages Router](#pages-router)
  - [App Router](#app-router)
  - [Error Handling](#error-handling)
  - [Response Adapter](#response-adapter)
  - [With Error Handler HOC](#with-error-handler-hoc)
- [API Reference](#api-reference)
- [Error Handling Reference](#error-handling-reference)
- [TypeScript Support](#typescript-support)
- [Compatibility Matrix](#compatibility-matrix)
- [Contributing](#contributing)
- [License](#license)
- [Author](#author)

## What is it?

`@syncafricabs/kernspark-nextjs` is a framework adapter that bridges the [SyncAfrica KernSpark Core](https://github.com/iamprovy-dev/kernspark-js/tree/main/packages/kernspark-core) with Next.js. It provides:

- **NextJsIntegration** - Maps `ApplicationError` instances to standardized JSON API responses for both App Router and Pages Router
- **NextJsResponseAdapter** - Helper methods for sending consistent success and error responses
- **withErrorHandler** - Higher-order function for wrapping route handlers with error handling
- **appRouterHandler** - Wrapper for App Router route handlers using the Web Request/Response API

The adapter follows the **adapter pattern**: it depends on `@syncafricabs/kernspark-core` (the framework-independent layer) and Next.js only in the adapter layer. The core has zero framework dependencies.

## Why it exists

Modern web applications benefit from a **KernSpark** - a common set of contracts, types, and behaviors shared across bounded contexts. The SyncAfrica KernSpark provides:

1. **Standardized API envelopes** (`ApiSuccess` / `ApiError`)
2. **Rich domain error hierarchy** (`ValidationError`, `BusinessError`, `AuthenticationError`, etc.)
3. **Result types** (`Result`, `Ok`, `Err`)
4. **Domain primitives** (`UUID`, `Money`, `Entity`, `ValueObject`)

Without adapters, each team would reimplement framework-specific plumbing to use these core types. This adapter eliminates that duplication by providing Next.js-specific bindings.

## Features

- **Pages Router support** - Drop-in API route error handler
- **App Router support** - Web-standard Request/Response wrapper
- **Standardized JSON responses** - All responses follow the `ApiSuccess` / `ApiError` envelope
- **Zero-configuration error handling** - Wraps route handlers automatically
- **Full TypeScript support** - Comprehensive type definitions included
- **Production-ready** - Proper error handling, stack traces, cause chains
- **Framework isolation** - Core has no Next.js dependencies

## Installation

```bash
npm install @syncafricabs/kernspark-nextjs @syncafricabs/kernspark-core next
```

Or with your preferred package manager:

```bash
yarn add @syncafricabs/kernspark-nextjs @syncafricabs/kernspark-core next
```

```bash
pnpm add @syncafricabs/kernspark-nextjs @syncafricabs/kernspark-core next
```

## Quick Start

### Pages Router

```typescript
// pages/api/users.ts
import type { NextApiRequest, NextApiResponse } from 'next';
import { withErrorHandler, NextJsResponseAdapter } from '@syncafricabs/kernspark-nextjs';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method === 'GET') {
    const user = findUser(req.query.id as string);
    if (!user) {
      throw new NotFoundError('User not found');
    }
    return NextJsResponseAdapter.ok(res, user, 'User retrieved successfully');
  }

  if (req.method === 'POST') {
    const { name, email } = req.body;
    if (!name || !email) {
      throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
    }
    const user = createUser(req.body);
    return NextJsResponseAdapter.created(res, user, 'User created successfully');
  }
}

export default withErrorHandler(handler);
```

### App Router

```typescript
// app/api/users/route.ts
import { appRouterHandler, NextJsResponseAdapter } from '@syncafricabs/kernspark-nextjs';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

export const GET = appRouterHandler(async ({ request }) => {
  const url = new URL(request.url);
  const id = url.searchParams.get('id');
  const user = findUser(id);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  return NextJsResponseAdapter.created({ status: 200, success: true, data: user, message: 'OK' });
});

export const POST = appRouterHandler(async ({ request }) => {
  const body = await request.json();
  if (!body.name || !body.email) {
    throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
  }
  const user = createUser(body);
  return NextJsResponseAdapter.created({ status: 201, success: true, data: user, message: 'Created' });
});
```

## Comprehensive Usage

### Pages Router

#### Basic API Route with Error Handling

```typescript
// pages/api/users/[id].ts
import type { NextApiRequest, NextApiResponse } from 'next';
import { withErrorHandler, NextJsResponseAdapter } from '@syncafricabs/kernspark-nextjs';
import {
  NotFoundError,
  ValidationError,
  ConflictError,
  AuthenticationError,
  InvalidTokenError,
  AuthorizationError,
  PermissionDeniedError,
} from '@syncafricabs/kernspark-core';

async function getUser(req: NextApiRequest, res: NextApiResponse) {
  const { id } = req.query;
  const user = findUser(id as string);

  if (!user) {
    throw new NotFoundError(`User with id ${id} not found`);
  }

  return NextJsResponseAdapter.ok(res, user, 'User retrieved successfully');
}

async function updateUser(req: NextApiRequest, res: NextApiResponse) {
  const { id } = req.query;
  const { name, email } = req.body;

  if (!name || !email) {
    throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
  }

  if (!isValidEmail(email)) {
    throw new ValidationError('INVALID_EMAIL', 'Email format is invalid');
  }

  const existingUser = findUserByEmail(email);
  if (existingUser && existingUser.id !== id) {
    throw new ConflictError('User with this email already exists');
  }

  const user = updateUserData(id as string, { name, email });
  return NextJsResponseAdapter.ok(res, user, 'User updated successfully');
}

async function deleteUser(req: NextApiRequest, res: NextApiResponse) {
  const token = req.headers.authorization;
  if (!token) {
    throw new InvalidTokenError('Missing authorization token');
  }

  if (!hasPermission(token, 'delete:users')) {
    throw new PermissionDeniedError('You do not have permission to delete users');
  }

  deleteUserData(req.query.id as string);
  return NextJsResponseAdapter.noContent(res, 'User deleted successfully');
}

export default withErrorHandler(
  (req: NextApiRequest, res: NextApiResponse) => {
    switch (req.method) {
      case 'GET':
        return getUser(req, res);
      case 'PUT':
        return updateUser(req, res);
      case 'DELETE':
        return deleteUser(req, res);
      default:
        return NextJsResponseAdapter.methodNotAllowed(res);
    }
  },
  {
    logErrors: true,
    includeCause: true,
    includeStack: process.env.NODE_ENV === 'development',
  }
);
```

#### Async Error Handling

```typescript
// pages/api/users.ts
import type { NextApiRequest, NextApiResponse } from 'next';
import { withErrorHandler, NextJsResponseAdapter } from '@syncafricabs/kernspark-nextjs';
import { NotFoundError } from '@syncafricabs/kernspark-core';

async function listUsers(req: NextApiRequest, res: NextApiResponse) {
  const users = await fetchUsersFromDatabase();
  return NextJsResponseAdapter.ok(res, users, 'Users retrieved successfully');
}

export default withErrorHandler(listUsers);
```

### App Router

#### Route Handler with Web API

```typescript
// app/api/users/route.ts
import { appRouterHandler, NextJsResponseAdapter } from '@syncafricabs/kernspark-nextjs';
import { NotFoundError, ValidationError, ConflictError } from '@syncafricabs/kernspark-core';

export const GET = appRouterHandler(async ({ request }) => {
  const url = new URL(request.url);
  const id = url.searchParams.get('id');
  const user = findUser(id);

  if (!user) {
    throw new NotFoundError('User not found');
  }

  return Response.json(
    { status: 200, success: true, data: user, message: 'User retrieved' },
    { status: 200 }
  );
});

export const POST = appRouterHandler(async ({ request }) => {
  const body = await request.json();

  if (!body.name || !body.email) {
    throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
  }

  if (findUserByEmail(body.email)) {
    throw new ConflictError('User with this email already exists');
  }

  const user = createUser(body);
  return Response.json(
    { status: 201, success: true, data: user, message: 'User created' },
    { status: 201 }
  );
});
```

#### Route Handler with Response Adapter

```typescript
// app/api/users/[id]/route.ts
import { appRouterHandler, NextJsResponseAdapter } from '@syncafricabs/kernspark-nextjs';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

export const GET = appRouterHandler(async ({ request }) => {
  const url = new URL(request.url);
  const id = url.pathname.split('/').pop();

  const user = findUser(id);
  if (!user) {
    throw new NotFoundError('User not found');
  }

  return NextJsResponseAdapter.created({ status: 200, success: true, data: user });
});
```

### Error Handling

#### Custom Error Handler Configuration

```typescript
import { createNextJsErrorHandler, NextJsErrorHandlerOptions } from '@syncafricabs/kernspark-nextjs';

const options: NextJsErrorHandlerOptions = {
  logErrors: true,
  logStack: true,
  includeCause: true,
  includeStack: process.env.NODE_ENV === 'development',
  defaultStatusCode: 500,
  logger: (error, req) => {
    console.error({
      correlationId: req.headers['x-correlation-id'],
      method: req.method,
      url: req.url,
      error: error.message,
      stack: error.stack,
    });
  },
};

export default withErrorHandler(handler, options);
```

#### Throwing Application Errors

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

// Validation errors
if (!name || !email) {
  throw new MissingFieldsError('Name and email are required');
}

if (!isValidEmail(email)) {
  throw new ValidationError('INVALID_EMAIL', 'Email format is invalid');
}

// Business errors
if (userExists(email)) {
  throw new ConflictError('User with this email already exists');
}

// Authentication errors
const token = req.headers.authorization;
if (!token || !isValidToken(token)) {
  throw new InvalidTokenError('Invalid or expired token');
}

// Authorization errors
if (!req.user?.canDeleteUsers) {
  throw new PermissionDeniedError('You do not have permission to delete users');
}

// Infrastructure errors
try {
  await sendEmail(user.email, 'Welcome');
} catch (error) {
  throw new ExternalServiceError('Failed to send welcome email', error as Error);
}
```

### Response Adapter

#### Success Responses

```typescript
import { NextJsResponseAdapter } from '@syncafricabs/kernspark-nextjs';

// 200 OK with data
NextJsResponseAdapter.ok(res, user, 'User retrieved');

// 201 Created with data
NextJsResponseAdapter.created(res, user, 'User created');

// 204 No Content
NextJsResponseAdapter.noContent(res);

// Send raw ApiSuccess
const success = { status: 200, success: true, data: users, message: 'OK' };
NextJsResponseAdapter.sendSuccess(res, success);
```

#### Error Responses

```typescript
import { NextJsResponseAdapter } from '@syncafricabs/kernspark-nextjs';

NextJsResponseAdapter.badRequest(res, 'MISSING_FIELDS', 'Email and password are required');
NextJsResponseAdapter.unauthorized(res, 'INVALID_CREDENTIALS', 'Invalid email or password');
NextJsResponseAdapter.forbidden(res, 'PERMISSION_DENIED', 'You do not have permission');
NextJsResponseAdapter.notFound(res, 'NOT_FOUND', 'User not found');
NextJsResponseAdapter.conflict(res, 'CONFLICT', 'Resource already exists');
NextJsResponseAdapter.unprocessableEntity(res, 'INVALID_DATA', 'Invalid request data');
NextJsResponseAdapter.tooManyRequests(res, 'RATE_LIMIT_EXCEEDED', 'Too many requests');
NextJsResponseAdapter.internalServerError(res, 'INTERNAL_ERROR', 'An unexpected error occurred');
NextJsResponseAdapter.badGateway(res, 'BAD_GATEWAY', 'Upstream service error');
NextJsResponseAdapter.serviceUnavailable(res, 'SERVICE_UNAVAILABLE', 'Service temporarily unavailable');
NextJsResponseAdapter.gatewayTimeout(res, 'GATEWAY_TIMEOUT', 'Upstream service timeout');
```

### With Error Handler HOC

The `withErrorHandler` function wraps API route handlers and automatically catches `ApplicationError` instances:

```typescript
import { withErrorHandler, NextJsResponseAdapter } from '@syncafricabs/kernspark-nextjs';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

async function handler(req: NextApiRequest, res: NextApiResponse) {
  const user = findUser(req.query.id as string);
  if (!user) {
    throw new NotFoundError('User not found');
  }
  return NextJsResponseAdapter.ok(res, user);
}

export default withErrorHandler(handler, {
  logErrors: true,
  includeCause: true,
  includeStack: process.env.NODE_ENV === 'development',
});
```

## API Reference

### `nextJsErrorHandler`

Next.js error handling middleware. Catches all errors and maps them to standardized JSON responses.

```typescript
function nextJsErrorHandler(error: Error, req: NextApiRequest, res: NextApiResponse): void;
```

### `createNextJsErrorHandler(options?: NextJsErrorHandlerOptions)`

Creates a customized error handler.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `logErrors` | `boolean` | `true` | Enable/disable error logging |
| `logStack` | `boolean` | `false` | Include stack traces in logs |
| `includeCause` | `boolean` | `true` | Include error cause in response |
| `includeStack` | `boolean` | `false` | Include stack traces in response (dev only) |
| `defaultStatusCode` | `number` | `500` | Default status for unknown errors |
| `logger` | `function` | `console.error` | Custom logger function |

### `NextJsResponseAdapter`

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

### `withErrorHandler(handler, options?)`

Wraps an API route handler with error handling.

```typescript
function withErrorHandler<T extends NextJsApiHandler>(
  handler: T,
  options?: NextJsErrorHandlerOptions
): NextJsApiHandler;
```

### `appRouterHandler(handler, options?)`

Wraps an App Router route handler with error handling.

```typescript
function appRouterHandler<T>(
  handler: (request: Request) => Promise<Response>,
  options?: AppRouterHandlerOptions
): (request: Request) => Promise<Response>;
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

Full TypeScript support with strict mode enabled. The adapter extends Next.js types:

```typescript
import { NextApiRequest, NextApiResponse } from 'next';

// Correlation ID can be accessed from headers
const correlationId = req.headers['x-correlation-id'];

// Type augmentation for response helpers
declare module '@syncafricabs/kernspark-nextjs' {
  export interface NextJsApiHandlerOptions {
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

| Package Version | Node.js | Next.js | @syncafricabs/kernspark-core |
|----------------|---------|---------|----------------------------------|
| 1.0.0 | >=18.0.0 | ^13.0.0 | ^1.0.0 |

| Feature | Next.js 13+ |
|---------|-------------|
| Pages Router API routes | Supported |
| App Router route handlers | Supported |
| Error handling | Supported |
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
cd kernspark/packages/kernspark-nextjs

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


