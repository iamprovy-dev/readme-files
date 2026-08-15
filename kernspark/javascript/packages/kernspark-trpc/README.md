# @syncafricabs/kernspark-trpc

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue)](https://www.typescriptlang.org/)
[![tRPC](https://img.shields.io/badge/tRPC-10%2B-3988CE)](https://trpc.io/)

tRPC adapter for the [SyncAfrica KernSpark](https://github.com/iamprovy-dev/kernspark-js) ecosystem. This package provides production-ready tRPC error formatters, middleware, and utilities that integrate the framework-independent `@syncafricabs/kernspark-core` with tRPC.

## Table of Contents

- [What is it?](#what-is-it)
- [Why it exists](#why-it-exists)
- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Comprehensive Usage](#comprehensive-usage)
  - [Error Formatter](#error-formatter)
  - [Middleware](#middleware)
  - [Response Adapter](#response-adapter)
  - [Complete Router Example](#complete-router-example)
- [API Reference](#api-reference)
- [Error Handling Reference](#error-handling-reference)
- [TypeScript Support](#typescript-support)
- [Compatibility Matrix](#compatibility-matrix)
- [Contributing](#contributing)
- [License](#license)
- [Author](#author)

## What is it?

`@syncafricabs/kernspark-trpc` is a framework adapter that bridges the [SyncAfrica KernSpark Core](https://github.com/iamprovy-dev/kernspark-js/tree/main/packages/kernspark-core) with tRPC. It provides:

- **TrpcIntegration** - Maps `ApplicationError` instances to standardized tRPC error responses
- **TrpcResponseAdapter** - Helper methods for creating consistent success and error responses
- **trpcErrorFormatter** - Custom tRPC error formatter with KernSpark error mapping
- **trpcMiddleware** - Pre-configured tRPC middleware with logging and correlation ID support

The adapter follows the **adapter pattern**: it depends on `@syncafricabs/kernspark-core` (the framework-independent layer) and tRPC only in the adapter layer. The core has zero framework dependencies.

## Why it exists

Modern web applications benefit from a **KernSpark** - a common set of contracts, types, and behaviors shared across bounded contexts. The SyncAfrica KernSpark provides:

1. **Standardized API envelopes** (`ApiSuccess` / `ApiError`)
2. **Rich domain error hierarchy** (`ValidationError`, `BusinessError`, `AuthenticationError`, etc.)
3. **Result types** (`Result`, `Ok`, `Err`)
4. **Domain primitives** (`UUID`, `Money`, `Entity`, `ValueObject`)

Without adapters, each team would reimplement framework-specific plumbing to use these core types. This adapter eliminates that duplication by providing tRPC-specific bindings.

## Features

- **Error formatter support** - Custom tRPC error formatter with KernSpark error mapping
- **Standardized error responses** - All errors follow the `ApiError` envelope
- **Middleware support** - Pre-configured tRPC middleware with logging
- **Full TypeScript support** - Comprehensive type definitions included
- **Production-ready** - Proper error handling, stack traces, cause chains
- **Framework isolation** - Core has no tRPC dependencies

## Installation

```bash
npm install @syncafricabs/kernspark-trpc @syncafricabs/kernspark-core @trpc/server
```

Or with your preferred package manager:

```bash
yarn add @syncafricabs/kernspark-trpc @syncafricabs/kernspark-core @trpc/server
```

```bash
pnpm add @syncafricabs/kernspark-trpc @syncafricabs/kernspark-core @trpc/server
```

## Quick Start

```typescript
import { initTRPC } from '@trpc/server';
import { trpcErrorFormatter, trpcMiddleware } from '@syncafricabs/kernspark-trpc';
import { NotFoundError, ValidationError } from '@syncafricabs/kernspark-core';

const t = initTRPC.middleware<{
  context: {};
}>({
  errorFormatter: trpcErrorFormatter,
});

const appRouter = t.router({
  getUser: t.procedure
    .input(z.object({ id: z.string() }))
    .query(({ input }) => {
      const user = findUser(input.id);
      if (!user) {
        throw new NotFoundError('User not found');
      }
      return TrpcResponseAdapter.sendSuccess(user, 'User retrieved');
    }),

  createUser: t.procedure
    .input(z.object({ name: z.string(), email: z.string() }))
    .mutation(({ input }) => {
      if (!input.name || !input.email) {
        throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
      }
      const user = createUser(input);
      return TrpcResponseAdapter.created(user, 'User created');
    }),
});
```

## Comprehensive Usage

### Error Formatter

#### Basic Error Formatter

```typescript
import { initTRPC } from '@trpc/server';
import { trpcErrorFormatter } from '@syncafricabs/kernspark-trpc';

const t = initTRPC.middleware<{
  context: {};
}>({
  errorFormatter: trpcErrorFormatter,
});
```

#### Custom Error Formatter Configuration

```typescript
import { createTrpcErrorFormatter, TrpcErrorFormatterOptions } from '@syncafricabs/kernspark-trpc';

const options: TrpcErrorFormatterOptions = {
  logErrors: true,
  logStack: true,
  includeCause: true,
  includeStack: process.env.NODE_ENV === 'development',
  defaultStatusCode: 500,
  logger: (error, path, input) => {
    console.error({
      path,
      input,
      error: error.message,
      stack: error.stack,
    });
  },
};

const t = initTRPC.middleware<{
  context: {};
}>({
  errorFormatter: createTrpcErrorFormatter(options),
});
```

### Middleware

#### Basic Middleware

```typescript
import { trpcMiddleware } from '@syncafricabs/kernspark-trpc';

const t = initTRPC.middleware<{
  context: {};
}>({
  middleware: trpcMiddleware,
});
```

#### Custom Middleware Configuration

```typescript
import { createTrpcMiddleware, TrpcMiddlewareOptions } from '@syncafricabs/kernspark-trpc';

const options: TrpcMiddlewareOptions = {
  logRequests: true,
  logResponses: true,
  correlationIdHeader: 'x-correlation-id',
  logger: (meta) => {
    console.log(JSON.stringify(meta));
  },
};

const t = initTRPC.middleware<{
  context: {};
}>({
  middleware: createTrpcMiddleware(options),
});
```

### Response Adapter

#### Success Responses

```typescript
import { TrpcResponseAdapter } from '@syncafricabs/kernspark-trpc';

// 200 OK with data
return TrpcResponseAdapter.sendSuccess(user, 'User retrieved');

// 201 Created with data
return TrpcResponseAdapter.created(user, 'User created');

// 204 No Content
return TrpcResponseAdapter.noContent();
```

#### Error Responses

```typescript
import { TrpcResponseAdapter } from '@syncafricabs/kernspark-trpc';

throw new Error(JSON.stringify(TrpcResponseAdapter.badRequest('MISSING_FIELDS', 'Email and password are required')));
throw new Error(JSON.stringify(TrpcResponseAdapter.unauthorized('INVALID_CREDENTIALS', 'Invalid email or password')));
throw new Error(JSON.stringify(TrpcResponseAdapter.forbidden('PERMISSION_DENIED', 'You do not have permission')));
throw new Error(JSON.stringify(TrpcResponseAdapter.notFound('NOT_FOUND', 'User not found')));
throw new Error(JSON.stringify(TrpcResponseAdapter.conflict('CONFLICT', 'Resource already exists')));
throw new Error(JSON.stringify(TrpcResponseAdapter.unprocessableEntity('INVALID_DATA', 'Invalid request data')));
throw new Error(JSON.stringify(TrpcResponseAdapter.tooManyRequests('RATE_LIMIT_EXCEEDED', 'Too many requests')));
throw new Error(JSON.stringify(TrpcResponseAdapter.internalServerError('INTERNAL_ERROR', 'An unexpected error occurred')));
throw new Error(JSON.stringify(TrpcResponseAdapter.badGateway('BAD_GATEWAY', 'Upstream service error')));
throw new Error(JSON.stringify(TrpcResponseAdapter.serviceUnavailable('SERVICE_UNAVAILABLE', 'Service temporarily unavailable')));
throw new Error(JSON.stringify(TrpcResponseAdapter.gatewayTimeout('GATEWAY_TIMEOUT', 'Upstream service timeout')));
```

### Complete Router Example

```typescript
import { initTRPC } from '@trpc/server';
import { z } from 'zod';
import { trpcErrorFormatter, trpcMiddleware, TrpcResponseAdapter } from '@syncafricabs/kernspark-trpc';
import { NotFoundError, ValidationError, ConflictError, InvalidTokenError, PermissionDeniedError } from '@syncafricabs/kernspark-core';

const t = initTRPC.middleware<{
  context: {};
}>({
  errorFormatter: trpcErrorFormatter,
  middleware: trpcMiddleware,
});

export const appRouter = t.router({
  health: t.procedure.query(() => {
    return TrpcResponseAdapter.sendSuccess({ status: 'healthy' }, 'Service is healthy');
  }),

  users: t.router({
    list: t.procedure
      .input(z.object({ page: z.number().optional(), limit: z.number().optional() }))
      .query(({ input }) => {
        const users = getUsers(input.page || 1, input.limit || 10);
        return TrpcResponseAdapter.sendSuccess(users, 'Users retrieved successfully');
      }),

    get: t.procedure
      .input(z.object({ id: z.string() }))
      .query(({ input }) => {
        const user = findUser(input.id);
        if (!user) {
          throw new NotFoundError('User not found');
        }
        return TrpcResponseAdapter.sendSuccess(user, 'User retrieved successfully');
      }),

    create: t.procedure
      .input(z.object({ name: z.string(), email: z.string() }))
      .mutation(({ input }) => {
        if (!input.name || !input.email) {
          throw new ValidationError('MISSING_FIELDS', 'Name and email are required');
        }

        if (findUserByEmail(input.email)) {
          throw new ConflictError('User with this email already exists');
        }

        const user = createUser(input);
        return TrpcResponseAdapter.created(user, 'User created successfully');
      }),

    delete: t.procedure
      .input(z.object({ id: z.string() }))
      .mutation(({ input, ctx }) => {
        const token = ctx.req?.headers?.authorization;
        if (!token || !isValidToken(token)) {
          throw new InvalidTokenError('Invalid or expired token');
        }

        const user = findUser(input.id);
        if (!user) {
          throw new NotFoundError('User not found');
        }

        if (!canDeleteUser(token, user.id)) {
          throw new PermissionDeniedError('You do not have permission to delete users');
        }

        deleteUser(user.id);
        return TrpcResponseAdapter.noContent('User deleted successfully');
      }),
  }),
});

export type AppRouter = typeof appRouter;
```

## API Reference

### `trpcErrorFormatter`

tRPC error formatter. Maps `ApplicationError` instances to standardized tRPC error responses.

```typescript
function trpcErrorFormatter(error: TRPCError | Error, path: string, input: any): TRPCError;
```

### `createTrpcErrorFormatter(options?: TrpcErrorFormatterOptions)`

Creates a customized error formatter.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `logErrors` | `boolean` | `true` | Enable/disable error logging |
| `logStack` | `boolean` | `false` | Include stack traces in logs |
| `includeCause` | `boolean` | `true` | Include error cause in response |
| `includeStack` | `boolean` | `false` | Include stack traces in response (dev only) |
| `defaultStatusCode` | `number` | `500` | Default status for unknown errors |
| `logger` | `function` | `console.error` | Custom logger function |

### `TrpcResponseAdapter`

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

### `trpcMiddleware`

Pre-configured tRPC middleware with logging and correlation ID support.

```typescript
function trpcMiddleware(opts: { ctx: any; path: string; input: any; type: string; next: () => Promise<any> }): Promise<any>;
```

### `createTrpcMiddleware(options?: TrpcMiddlewareOptions)`

Creates a customized tRPC middleware.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `logRequests` | `boolean` | `true` | Log incoming requests |
| `logResponses` | `boolean` | `true` | Log outgoing responses |
| `correlationIdHeader` | `string` | `'x-correlation-id'` | Header name for correlation ID |
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

Full TypeScript support with strict mode enabled. The adapter extends tRPC types:

```typescript
import { initTRPC } from '@trpc/server';
import type { AnyRouter } from '@trpc/server';

// Router type augmentation
interface AppRouter {
  health: {
    query: {}[];
    output: { status: string; message: string };
  };
  users: {
    list: {
      input: { page?: number; limit?: number };
      output: { users: User[]; total: number };
    };
    get: {
      input: { id: string };
      output: User;
    };
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

| Package Version | Node.js | tRPC | @syncafricabs/kernspark-core |
|----------------|---------|------|----------------------------------|
| 1.0.0 | >=18.0.0 | ^10.0.0 | ^1.0.0 |

| Feature | tRPC 10+ |
|---------|-------------|
| Error formatter | Supported |
| Middleware | Supported |
| Router | Supported |
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
cd kernspark/packages/kernspark-trpc

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


