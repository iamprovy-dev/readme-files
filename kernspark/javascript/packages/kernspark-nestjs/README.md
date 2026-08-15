# @syncafricabs/kernspark-nestjs

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.0-blue)](https://www.typescriptlang.org/)
[![NestJS](https://img.shields.io/badge/NestJS-9.x%20%7C%2010.x-green)](https://nestjs.com/)

NestJS adapter for the [SyncAfrica KernSpark](https://github.com/iamprovy-dev/kernspark-js) ecosystem. This package provides production-ready NestJS exception filters, interceptors, and module definitions that integrate the framework-independent `@syncafricabs/kernspark-core` with NestJS.

## Table of Contents

- [What is it?](#what-is-it)
- [Why it exists](#why-it-exists)
- [Features](#features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Comprehensive Usage](#comprehensive-usage)
  - [Exception Filter](#exception-filter)
  - [Response Interceptor](#response-interceptor)
  - [Module Setup](#module-setup)
- [API Reference](#api-reference)
- [Error Handling Reference](#error-handling-reference)
- [TypeScript Support](#typescript-support)
- [Compatibility Matrix](#compatibility-matrix)
- [Contributing](#contributing)
- [License](#license)

## What is it?

`@syncafricabs/kernspark-nestjs` is a framework adapter that bridges the [SyncAfrica KernSpark Core](https://github.com/iamprovy-dev/kernspark-js/tree/main/packages/kernspark-core) with NestJS. It provides:

- **exceptions** - Global exception filter that catches `ApplicationError` instances and converts them to standardized JSON API responses
- **NestJsResponseInterceptor** - Response interceptor that wraps successful responses in the `ApiSuccess` envelope
- **NestJsModule** - NestJS module definition that registers filters and interceptors globally

The adapter follows the **adapter pattern**: it depends on `@syncafricabs/kernspark-core` (the framework-independent layer) and NestJS only in the adapter layer. The core has zero framework dependencies.

## Why it exists

Modern microservice architectures benefit from a **KernSpark** - a common set of contracts, types, and behaviors shared across bounded contexts. The SyncAfrica KernSpark provides:

1. **Standardized API envelopes** (`ApiSuccess` / `ApiError`)
2. **Rich domain error hierarchy** (`ValidationError`, `BusinessError`, `AuthenticationError`, etc.)
3. **Result types** (`Result`, `Ok`, `Err`)
4. **Domain primitives** (`UUID`, `Money`, `Entity`, `ValueObject`)

Without adapters, each team would reimplement framework-specific plumbing to use these core types. This adapter eliminates that duplication by providing NestJS-specific bindings.

## Features

- **Global exception filtering** - Catch all `ApplicationError` instances with a single filter
- **Standardized JSON responses** - All responses follow the `ApiSuccess` / `ApiError` envelope
- **Response interception** - Automatically wrap successful responses
- **Module-based registration** - Clean NestJS module integration
- **Full TypeScript support** - Comprehensive type definitions with decorators
- **Production-ready** - Proper error handling, stack traces, cause chains
- **Framework isolation** - Core has no NestJS dependencies

## Installation

```bash
npm install @syncafricabs/kernspark-nestjs @syncafricabs/kernspark-core @nestjs/common @nestjs/core @nestjs/platform-express reflect-metadata rxjs
```

Or with your preferred package manager:

```bash
yarn add @syncafricabs/kernspark-nestjs @syncafricabs/kernspark-core @nestjs/common @nestjs/core @nestjs/platform-express reflect-metadata rxjs
```

## Quick Start

```typescript
import { Module } from '@nestjs/common';
import { APP_FILTER, APP_INTERCEPTOR } from '@nestjs/core';
import { exceptions, NestJsResponseInterceptor, KernsparkNestJsModule } from '@syncafricabs/kernspark-nestjs';
import { ValidationError, NotFoundError } from '@syncafricabs/kernspark-core';

@Module({
  imports: [
    KernsparkNestJsModule.forRoot({
      filterOptions: { logErrors: true, includeStack: false },
      interceptorOptions: { defaultMessage: 'OK', envelopeSuccess: true },
      global: true,
    }),
  ],
  providers: [
    {
      provide: APP_FILTER,
      useClass: exceptions,
    },
    {
      provide: APP_INTERCEPTOR,
      useClass: NestJsResponseInterceptor,
    },
  ],
})
export class AppModule {}
```

```typescript
import { Controller, Get, Param, Post, Body, Inject } from '@nestjs/common';
import { NotFoundError, ConflictError } from '@syncafricabs/kernspark-core';

@Controller('users')
export class UsersController {
  @Get(':id')
  findOne(@Param('id') id: string) {
    const user = findUser(id);
    if (!user) {
      throw new NotFoundError('User not found');
    }
    return user;
  }

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    if (userExists(createUserDto.email)) {
      throw new ConflictError('User already exists');
    }
    return createUser(createUserDto);
  }
}
```

## Comprehensive Usage

### Exception Filter

The `exceptions` is a global exception filter that catches all unhandled exceptions and maps them to standardized JSON responses following the `ApiError` envelope.

#### Basic Setup

```typescript
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';
import { exceptions } from '@syncafricabs/kernspark-nestjs';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: exceptions,
    },
  ],
})
export class AppModule {}
```

#### Custom Exception Filter Configuration

```typescript
import { exceptions, exceptionsOptions } from '@syncafricabs/kernspark-nestjs';

const options: exceptionsOptions = {
  logErrors: true,
  logStack: true,
  includeCause: true,
  includeStack: process.env.NODE_ENV === 'development',
  defaultStatusCode: 500,
  logger: (error, req) => {
    console.error({
      correlationId: req.headers['x-correlation-id'],
      method: req.method,
      url: req.originalUrl,
      error: error.message,
      stack: error.stack,
    });
  },
};

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useValue: new exceptions(options),
    },
  ],
})
export class AppModule {}
```

#### Throwing Errors from Controllers

```typescript
import {
  Controller,
  Get,
  Post,
  Body,
  Param,
  UseGuards,
  Request,
} from '@nestjs/common';
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

@Controller('users')
export class UsersController {
  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    const { name, email } = createUserDto;

    if (!name || !email) {
      throw new MissingFieldsError('Name and email are required');
    }

    if (!isValidEmail(email)) {
      throw new ValidationError('INVALID_EMAIL', 'Email format is invalid');
    }

    if (userExists(email)) {
      throw new ConflictError('User with this email already exists');
    }

    const user = createUser(createUserDto);
    return user;
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    const user = findUser(id);
    if (!user) {
      throw new NotFoundError('User not found');
    }
    return user;
  }

  @Delete(':id')
  @UseGuards(JwtAuthGuard)
  remove(@Request() req, @Param('id') id: string) {
    if (!req.user) {
      throw new InvalidTokenError('Invalid or expired token');
    }

    if (!req.user.canDeleteUsers) {
      throw new PermissionDeniedError('You do not have permission to delete users');
    }

    deleteUser(id);
    return { message: 'User deleted successfully' };
  }

  @Get('external/:id')
  async fetchExternal(@Param('id') id: string) {
    try {
      const data = await externalService.fetch(id);
      return data;
    } catch (error) {
      throw new ExternalServiceError('Failed to fetch data from external service');
    }
  }
}
```

### Response Interceptor

The `NestJsResponseInterceptor` wraps all successful responses in the `ApiSuccess` envelope, ensuring consistent response formatting across your API.

#### Basic Setup

```typescript
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';
import { NestJsResponseInterceptor } from '@syncafricabs/kernspark-nestjs';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: NestJsResponseInterceptor,
    },
  ],
})
export class AppModule {}
```

#### Custom Interceptor Configuration

```typescript
import { NestJsResponseInterceptor, NestJsResponseInterceptorOptions } from '@syncafricabs/kernspark-nestjs';

const options: NestJsResponseInterceptorOptions = {
  defaultMessage: 'Success',
  envelopeSuccess: true,
};

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useValue: new NestJsResponseInterceptor(options),
    },
  ],
})
export class AppModule {}
```

#### Bypassing the Interceptor

```typescript
import { Controller, Get, UseInterceptors } from '@nestjs/common';
import { NestJsResponseInterceptor } from '@syncafricabs/kernspark-nestjs';
import { FileInterceptor } from '@nestjs/platform-express';

@Controller('files')
export class FilesController {
  @Get('raw')
  @UseInterceptors(FileInterceptor('file'))
  getRawFile() {
    return rawBuffer;
  }

  @Get('stream')
  @UseInterceptors(StreamInterceptor)
  getStream() {
    return stream;
  }
}

class StreamInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    return next.handle();
  }
}
```

### Module Setup

The `KernsparkNestJsModule` provides a clean way to register all KernSpark functionality in your NestJS application.

#### Synchronous Module Registration

```typescript
import { Module } from '@nestjs/common';
import { KernsparkNestJsModule } from '@syncafricabs/kernspark-nestjs';

@Module({
  imports: [
    KernsparkNestJsModule.forRoot({
      filterOptions: {
        logErrors: true,
        logStack: process.env.NODE_ENV === 'development',
        includeCause: true,
        includeStack: process.env.NODE_ENV === 'development',
      },
      interceptorOptions: {
        defaultMessage: 'OK',
        envelopeSuccess: true,
      },
      global: true,
    }),
  ],
})
export class AppModule {}
```

#### Asynchronous Module Registration

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { KernsparkNestJsModule } from '@syncafricabs/kernspark-nestjs';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    KernsparkNestJsModule.forRootAsync({
      useFactory: (configService: ConfigService) => ({
        filterOptions: {
          logErrors: configService.get('NODE_ENV') === 'development',
          logStack: configService.get('NODE_ENV') === 'development',
          includeCause: true,
          includeStack: configService.get('NODE_ENV') === 'development',
          logger: (error, req) => {
            console.error({
              correlationId: req.headers['x-correlation-id'],
              method: req.method,
              url: req.originalUrl,
              error: error.message,
            });
          },
        },
        interceptorOptions: {
          defaultMessage: configService.get('API_DEFAULT_MESSAGE', 'OK'),
          envelopeSuccess: true,
        },
        global: true,
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

#### Feature Module Integration

```typescript
import { Module } from '@nestjs/common';
import { KernsparkNestJsModule } from '@syncafricabs/kernspark-nestjs';
import { UsersModule } from './users/users.module';

@Module({
  imports: [
    KernsparkNestJsModule.forRoot({
      filterOptions: { logErrors: true },
      interceptorOptions: { envelopeSuccess: true },
      global: false,
    }),
    UsersModule,
  ],
})
export class AppModule {}
```

## API Reference

### `exceptions`

Global exception filter implementing NestJS `ExceptionFilter` interface.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `logErrors` | `boolean` | `true` | Enable/disable error logging |
| `logStack` | `boolean` | `false` | Include stack traces in logs |
| `includeCause` | `boolean` | `true` | Include error cause in response |
| `includeStack` | `boolean` | `false` | Include stack traces in response (dev only) |
| `defaultStatusCode` | `number` | `500` | Default status for unknown errors |
| `logger` | `function` | `console.error` | Custom logger function |

**Response Format:**

```json
{
  "status": 400,
  "success": false,
  "errorCode": "MISSING_FIELDS",
  "message": "Name and email are required",
  "data": { "cause": "Validation chain failed" }
}
```

### `NestJsResponseInterceptor`

NestJS interceptor that wraps successful responses in `ApiSuccess` envelope.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `defaultMessage` | `string` | `'OK'` | Default message for responses |
| `envelopeSuccess` | `boolean` | `true` | Wrap responses in envelope |

**Response Format (enabled):**

```json
{
  "status": 200,
  "success": true,
  "message": "OK",
  "data": { "id": 1, "name": "John" }
}
```

**Response Format (disabled):**

```json
{
  "id": 1,
  "name": "John"
}
```

### `KernsparkNestJsModule`

NestJS module for registering KernSpark functionality.

#### `forRoot(options)`

Synchronous module registration.

```typescript
KernsparkNestJsModule.forRoot({
  filterOptions?: exceptionsOptions,
  interceptorOptions?: NestJsResponseInterceptorOptions,
  global?: boolean,
});
```

#### `forRootAsync(options)`

Asynchronous module registration with dependency injection.

```typescript
KernsparkNestJsModule.forRootAsync({
  useFactory: (configService: ConfigService) => ({
    filterOptions: { /* ... */ },
    interceptorOptions: { /* ... */ },
  }),
  inject: [ConfigService],
});
```

## Error Handling Reference

The exception filter maps the following `ApplicationError` subclasses:

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

Unknown `ApplicationError` instances use their embedded `statusCode`. NestJS `HttpException` instances are preserved with their original status code and message.

## TypeScript Support

Full TypeScript support with strict mode enabled and decorator metadata support.

```typescript
import { Module } from '@nestjs/common';
import { exceptions, NestJsResponseInterceptor } from '@syncafricabs/kernspark-nestjs';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: exceptions,
    },
    {
      provide: APP_INTERCEPTOR,
      useClass: NestJsResponseInterceptor,
    },
  ],
})
export class AppModule {}
```

### Decorator Metadata

Ensure your `tsconfig.json` includes:

```json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

## Compatibility Matrix

| Package Version | Node.js | NestJS | @syncafricabs/kernspark-core |
|----------------|---------|--------|----------------------------------|
| 1.0.0 | >=18.0.0 | ^9.0.0 \|\| ^10.0.0 | ^1.0.0 |

| Feature | NestJS 9.x | NestJS 10.x |
|---------|------------|-------------|
| Exception Filter | Supported | Supported |
| Response Interceptor | Supported | Supported |
| Dynamic Module | Supported | Supported |
| Async Configuration | Supported | Supported |
| TypeScript 5.0+ | Supported | Supported |

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
cd kernspark/packages/kernspark-nestjs

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


