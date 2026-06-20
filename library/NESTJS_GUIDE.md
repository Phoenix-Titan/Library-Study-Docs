# NestJS Complete Guide (v11 — 2026 Edition)

A full, study-friendly reference for **NestJS 11** — the progressive Node.js framework for building efficient, scalable, and enterprise-grade server-side applications.

> **What it is:** NestJS is a TypeScript-first framework that wraps Express (or Fastify) and brings Angular-style architecture — Modules, Controllers, Providers — to the backend. It gives you a battle-tested structure out of the box so teams write consistent, testable code from day one.
> **Why use it:** When you need a real backend — REST APIs, WebSockets, microservices, GraphQL — and you want the discipline of a structured framework rather than a pile of plain Express middleware. NestJS pays for itself on any project larger than a toy.

> **Version note:** This guide targets **NestJS 11** (released early 2026). NestJS 11 requires Node.js 20+ and TypeScript 5.x. Where APIs changed recently, look for the **⚡ Version note** flag.

---

## Table of Contents
1. [What NestJS Is, When to Use It & Architecture Overview](#1-what-nestjs-is-when-to-use-it--architecture-overview)
2. [Setup & the Nest CLI](#2-setup--the-nest-cli)
3. [Project & Folder Structure](#3-project--folder-structure)
4. [Modules](#4-modules)
5. [Controllers & Routing](#5-controllers--routing)
6. [Providers & Dependency Injection](#6-providers--dependency-injection)
7. [DTOs & Validation](#7-dtos--validation)
8. [Pipes, Guards, Interceptors, Middleware & Exception Filters](#8-pipes-guards-interceptors-middleware--exception-filters)
9. [Configuration (@nestjs/config)](#9-configuration-nestjsconfig)
10. [Database Integration — TypeORM, Prisma & Mongoose](#10-database-integration--typeorm-prisma--mongoose)
11. [Authentication & Authorization (Passport + JWT)](#11-authentication--authorization-passport--jwt)
12. [OpenAPI / Swagger Docs](#12-openapi--swagger-docs)
13. [WebSockets & Gateways](#13-websockets--gateways)
14. [File Uploads](#14-file-uploads)
15. [Caching, Task Scheduling & Queues](#15-caching-task-scheduling--queues)
16. [Microservices Overview](#16-microservices-overview)
17. [Testing — Unit & E2E](#17-testing--unit--e2e)
18. [Tips, Tricks & Gotchas](#18-tips-tricks--gotchas)
19. [Study Path](#19-study-path)

---

## 1. What NestJS Is, When to Use It & Architecture Overview

### What it is

NestJS is **not** a replacement for Express or Fastify — it sits **on top** of them as an abstraction layer. By default it uses Express under the hood; you can swap to Fastify for a performance boost with one config change.

What NestJS adds:
- **TypeScript-first design** — decorators, interfaces, and types everywhere.
- **Opinionated structure** — Modules, Controllers, Providers mirror Angular's architecture.
- **Built-in Dependency Injection (DI) container** — services wire themselves together automatically.
- **Lifecycle hooks** — `OnModuleInit`, `OnApplicationBootstrap`, `OnModuleDestroy`, etc.
- **Official packages** for nearly everything: auth, config, caching, queues, WebSockets, Swagger, GraphQL.

### When to use NestJS

| Situation | Use NestJS? |
|-----------|-------------|
| REST or GraphQL API for a SaaS product | Yes — ideal |
| Microservices with message brokers (Redis, RabbitMQ, Kafka) | Yes — first-class support |
| Real-time features (chat, notifications) via WebSockets | Yes |
| Simple one-route proxy / glue script | Overkill — use plain Express |
| CLI tool | Overkill |
| Large team needing consistent conventions | Strongly yes |

### Architecture overview

```
HTTP Request
     │
     ▼
┌──────────────────────────────────────────────┐
│  Middleware  (runs first — like Express MW)  │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│  Guards  (can I access this route?)          │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│  Interceptors (before handler — transform)   │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│  Pipes  (validate / transform incoming data) │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│  Controller Handler  (your route method)     │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│  Interceptors (after handler — transform)    │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────┐
│  Exception Filters  (catch errors anywhere)  │
└──────────────────────────────────────────────┘
```

The **Module** is the unit of organization. Every feature lives in a module. Modules import other modules, export providers, and the root `AppModule` ties it all together.

---

## 2. Setup & the Nest CLI

### Prerequisites

```bash
node -v   # must be >= 20
npm -v    # npm 9+ recommended
```

### Install the Nest CLI globally

```bash
npm install -g @nestjs/cli
nest --version   # should print 11.x.x
```

### Create a new project

```bash
nest new my-api
# CLI asks: which package manager? (npm / yarn / pnpm)
# Pick npm for simplicity, pnpm for monorepos

cd my-api
npm run start:dev   # watches for changes, restarts automatically
```

The app starts on `http://localhost:3000` by default.

### Nest CLI generate commands (schematics)

The `nest generate` (alias `nest g`) command creates files and wires them up automatically.

```bash
# Shorthand reference table
nest g module    users          # src/users/users.module.ts
nest g controller users         # src/users/users.controller.ts (auto-added to module)
nest g service   users          # src/users/users.service.ts   (auto-added to module)
nest g resource  users          # ALL of the above + DTOs + CRUD stub (fastest way to start)
nest g guard     auth           # src/auth/auth.guard.ts
nest g pipe      validation     # src/validation/validation.pipe.ts
nest g interceptor logging      # src/logging/logging.interceptor.ts
nest g middleware logger        # src/logger/logger.middleware.ts
nest g filter    http-exception # src/http-exception/http-exception.filter.ts
nest g gateway   events         # src/events/events.gateway.ts
```

> **Tip:** Add `--no-spec` to skip the auto-generated `.spec.ts` test file if you're scaffolding quickly.

```bash
nest g service users --no-spec
```

### Switching to Fastify (optional)

```bash
npm install @nestjs/platform-fastify fastify
```

```typescript
// main.ts — swap the adapter
import { NestFactory } from '@nestjs/core';
import { FastifyAdapter, NestFastifyApplication } from '@nestjs/platform-fastify';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter(),
  );
  await app.listen(3000, '0.0.0.0');
}
bootstrap();
```

> **⚡ Version note:** NestJS 11 ships with first-class Fastify v5 support. Some community middleware packages are Express-only — always check compatibility before switching.

---

## 3. Project & Folder Structure

The `nest new` scaffold produces this layout:

```
my-api/
├── src/
│   ├── main.ts                  # bootstrap: creates the Nest app, starts listening
│   ├── app.module.ts            # root module — imports all feature modules
│   ├── app.controller.ts        # root controller (usually delete this in real apps)
│   ├── app.controller.spec.ts   # root controller unit test
│   └── app.service.ts           # root service (usually delete this in real apps)
├── test/
│   ├── app.e2e-spec.ts          # end-to-end test
│   └── jest-e2e.json            # jest config for e2e
├── node_modules/
├── dist/                        # compiled JS output (git-ignored)
├── .env                         # environment variables (git-ignored!)
├── nest-cli.json                # Nest CLI config (source root, assets, etc.)
├── tsconfig.json                # TypeScript compiler config
├── tsconfig.build.json          # TS config used for build (excludes test files)
└── package.json
```

### Recommended scalable structure for real apps

As you add features, organize by **domain/feature**, not by layer:

```
src/
├── main.ts
├── app.module.ts
│
├── config/                      # app-wide config
│   └── configuration.ts
│
├── common/                      # shared building blocks
│   ├── decorators/
│   ├── filters/
│   │   └── http-exception.filter.ts
│   ├── guards/
│   ├── interceptors/
│   │   └── transform.interceptor.ts
│   └── pipes/
│
├── users/                       # feature module
│   ├── users.module.ts
│   ├── users.controller.ts
│   ├── users.service.ts
│   ├── users.repository.ts      # optional data-access layer
│   ├── dto/
│   │   ├── create-user.dto.ts
│   │   └── update-user.dto.ts
│   └── entities/
│       └── user.entity.ts
│
├── auth/                        # feature module
│   ├── auth.module.ts
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   ├── strategies/
│   │   └── jwt.strategy.ts
│   └── guards/
│       └── jwt-auth.guard.ts
│
└── database/                    # database module (TypeORM or Prisma)
    └── database.module.ts
```

---

## 4. Modules

Modules are the **structural unit** of NestJS. Every class belongs to a module.

### The `@Module()` decorator

```typescript
// users/users.module.ts
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  imports: [],          // other modules whose exports this module needs
  controllers: [UsersController], // controllers that belong to THIS module
  providers: [UsersService],      // services/providers that belong to THIS module
  exports: [UsersService],        // what THIS module makes available to importers
})
export class UsersModule {}
```

### Root (App) Module

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { UsersModule } from './users/users.module';
import { AuthModule } from './auth/auth.module';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }), // available everywhere, no re-import needed
    UsersModule,
    AuthModule,
  ],
})
export class AppModule {}
```

### Global Modules

Mark a module `@Global()` to skip importing it in every feature module. Use sparingly — usually only for config, logging, or database connections.

```typescript
import { Module, Global } from '@nestjs/common';
import { DatabaseService } from './database.service';

@Global()   // providers exported here are available everywhere
@Module({
  providers: [DatabaseService],
  exports: [DatabaseService],
})
export class DatabaseModule {}
```

### Dynamic Modules

Dynamic modules are configured at import time — the pattern used by `TypeOrmModule.forRoot()`, `ConfigModule.forRoot()`, etc.

```typescript
// database/database.module.ts — a simplified dynamic module example
import { Module, DynamicModule } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({})
export class DatabaseModule {
  static forRoot(options: { url: string }): DynamicModule {
    return {
      module: DatabaseModule,
      imports: [
        TypeOrmModule.forRoot({
          type: 'postgres',
          url: options.url,
          autoLoadEntities: true,
          synchronize: false, // NEVER true in production
        }),
      ],
      exports: [TypeOrmModule],
    };
  }
}

// Usage in app.module.ts:
// DatabaseModule.forRoot({ url: process.env.DATABASE_URL })
```

---

## 5. Controllers & Routing

Controllers handle incoming HTTP requests and return responses. They are **thin** — delegate actual work to services.

### Basic controller

```typescript
// users/users.controller.ts
import {
  Controller,
  Get,
  Post,
  Put,
  Patch,
  Delete,
  Param,
  Body,
  Query,
  HttpCode,
  HttpStatus,
  ParseIntPipe,
} from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@Controller('users')  // base path: /users
export class UsersController {
  // DI: NestJS injects UsersService automatically
  constructor(private readonly usersService: UsersService) {}

  // GET /users
  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  // GET /users/search?name=Alice&role=admin
  @Get('search')
  search(
    @Query('name') name: string,       // single query param
    @Query('role') role: string,
  ) {
    return this.usersService.search({ name, role });
  }

  // GET /users/:id
  @Get(':id')
  findOne(
    @Param('id', ParseIntPipe) id: number, // ParseIntPipe converts string→number, throws 400 if NaN
  ) {
    return this.usersService.findOne(id);
  }

  // POST /users — returns 201 by default for POST
  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  // PATCH /users/:id — partial update
  @Patch(':id')
  update(
    @Param('id', ParseIntPipe) id: number,
    @Body() updateUserDto: UpdateUserDto,
  ) {
    return this.usersService.update(id, updateUserDto);
  }

  // PUT /users/:id — full replacement
  @Put(':id')
  replace(
    @Param('id', ParseIntPipe) id: number,
    @Body() updateUserDto: UpdateUserDto,
  ) {
    return this.usersService.replace(id, updateUserDto);
  }

  // DELETE /users/:id — return 204 No Content
  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  remove(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.remove(id);
  }
}
```

### Common request decorators

| Decorator | What it does |
|-----------|-------------|
| `@Param('id')` | Route param: `/users/:id` → `id` |
| `@Query('page')` | Query string: `?page=2` → `page` |
| `@Body()` | Full request body (JSON) |
| `@Body('email')` | Single field from body |
| `@Headers('authorization')` | Single header value |
| `@Req()` | Raw Express/Fastify request object |
| `@Res()` | Raw response object (avoid when possible) |
| `@Ip()` | Client IP address |
| `@HostParam()` | Host-based routing param |

### Response status and headers

```typescript
import { Res, Header, HttpCode, HttpStatus } from '@nestjs/common';
import { Response } from 'express';

@Get('download')
@Header('Content-Type', 'application/pdf')        // set a static header
@HttpCode(HttpStatus.OK)                           // override default status
download(@Res({ passthrough: true }) res: Response) {
  // passthrough: true lets NestJS still handle the response lifecycle
  res.setHeader('Content-Disposition', 'attachment; filename="report.pdf"');
  return streamableFile; // or Buffer
}
```

> **Gotcha:** If you inject `@Res()` WITHOUT `passthrough: true`, NestJS hands full control to you — interceptors and the automatic JSON serialization won't work. Always add `{ passthrough: true }` unless you need raw control.

### Route wildcards and sub-routes

```typescript
@Get('ab*cd')          // matches abcd, ab_cd, ab123cd
@Get('profile/me')     // specific sub-path (order matters — put before :id)
@Get(':category/:id')  // multiple params
```

---

## 6. Providers & Dependency Injection

**Providers** are classes decorated with `@Injectable()`. They can be injected anywhere via the DI container. Services, repositories, factories, helpers — all are providers.

### A basic service

```typescript
// users/users.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { CreateUserDto } from './dto/create-user.dto';

// Temporary in-memory "database" for illustration
interface User {
  id: number;
  name: string;
  email: string;
}

@Injectable()
export class UsersService {
  private users: User[] = [];
  private nextId = 1;

  findAll(): User[] {
    return this.users;
  }

  findOne(id: number): User {
    const user = this.users.find((u) => u.id === id);
    if (!user) {
      // NestJS built-in exceptions set the correct HTTP status automatically
      throw new NotFoundException(`User #${id} not found`);
    }
    return user;
  }

  create(dto: CreateUserDto): User {
    const user: User = { id: this.nextId++, ...dto };
    this.users.push(user);
    return user;
  }

  update(id: number, dto: Partial<CreateUserDto>): User {
    const user = this.findOne(id);
    Object.assign(user, dto);
    return user;
  }

  remove(id: number): void {
    const idx = this.users.findIndex((u) => u.id === id);
    if (idx === -1) throw new NotFoundException(`User #${id} not found`);
    this.users.splice(idx, 1);
  }
}
```

### Custom providers

When you need more than just a class, use custom provider syntax in the module's `providers` array.

```typescript
// Value provider — inject a constant
{ provide: 'API_KEY', useValue: process.env.API_KEY }

// Factory provider — computed value, can inject other providers
{
  provide: 'DB_CONNECTION',
  useFactory: async (configService: ConfigService) => {
    const url = configService.get('DATABASE_URL');
    return createConnection(url); // async factory is supported
  },
  inject: [ConfigService], // dependencies for the factory
}

// Class provider — alias one class to another
{ provide: UsersRepository, useClass: MockUsersRepository }

// Existing provider alias
{ provide: 'ALIAS', useExisting: UsersService }
```

Injecting a custom provider by token:

```typescript
import { Inject, Injectable } from '@nestjs/common';

@Injectable()
export class AppService {
  constructor(
    @Inject('API_KEY') private readonly apiKey: string,
    @Inject('DB_CONNECTION') private readonly db: any,
  ) {}
}
```

### Provider scopes

By default, providers are **singletons** — one instance per application. You can change this:

```typescript
import { Injectable, Scope } from '@nestjs/common';

// DEFAULT — one instance for the whole app (shared)
@Injectable({ scope: Scope.DEFAULT })
export class SingletonService {}

// REQUEST — new instance per HTTP request (useful for request-scoped data like tenancy)
@Injectable({ scope: Scope.REQUEST })
export class RequestScopedService {}

// TRANSIENT — new instance every time it's injected
@Injectable({ scope: Scope.TRANSIENT })
export class TransientService {}
```

> **Gotcha:** REQUEST scope propagates up the dependency tree — any provider that injects a REQUEST-scoped provider also becomes REQUEST-scoped. This has performance implications. Prefer DEFAULT scope unless you have a specific need.

### Built-in HTTP exceptions

Throw these anywhere; NestJS maps them to the correct status code:

```typescript
import {
  BadRequestException,       // 400
  UnauthorizedException,     // 401
  ForbiddenException,        // 403
  NotFoundException,         // 404
  ConflictException,         // 409
  UnprocessableEntityException, // 422
  InternalServerErrorException, // 500
  ServiceUnavailableException,  // 503
} from '@nestjs/common';

throw new NotFoundException('User not found');
throw new ConflictException({ message: 'Email already exists', field: 'email' });
```

---

## 7. DTOs & Validation

**DTOs (Data Transfer Objects)** define the shape of incoming data. Combined with `class-validator` and `ValidationPipe`, they give you automatic, declarative validation.

### Install dependencies

```bash
npm install class-validator class-transformer
```

### Create a DTO

```typescript
// users/dto/create-user.dto.ts
import {
  IsEmail,
  IsString,
  IsNotEmpty,
  MinLength,
  MaxLength,
  IsOptional,
  IsEnum,
  IsInt,
  Min,
  Max,
  IsArray,
  ValidateNested,
} from 'class-validator';
import { Type } from 'class-transformer';

export enum UserRole {
  ADMIN = 'admin',
  USER = 'user',
  MODERATOR = 'moderator',
}

export class CreateUserDto {
  @IsString()
  @IsNotEmpty()
  @MinLength(2)
  @MaxLength(50)
  name: string;

  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8, { message: 'Password must be at least 8 characters' })
  password: string;

  @IsEnum(UserRole)
  @IsOptional()
  role?: UserRole = UserRole.USER;

  @IsInt()
  @Min(13)
  @Max(120)
  @IsOptional()
  age?: number;
}
```

### Update DTO with PartialType

`@nestjs/mapped-types` provides `PartialType` to make all fields optional (for PATCH endpoints):

```bash
npm install @nestjs/mapped-types
```

```typescript
// users/dto/update-user.dto.ts
import { PartialType } from '@nestjs/mapped-types';
import { CreateUserDto } from './create-user.dto';

// All fields from CreateUserDto become optional automatically
export class UpdateUserDto extends PartialType(CreateUserDto) {}
```

Other helpers: `OmitType`, `PickType`, `IntersectionType`.

### Enable the ValidationPipe globally

Apply once in `main.ts` — it runs for every request:

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Global validation pipe — ALWAYS do this
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,         // strip properties not in the DTO (security!)
      forbidNonWhitelisted: true, // throw 400 if extra properties are sent
      transform: true,         // auto-transform payloads to DTO class instances
      transformOptions: {
        enableImplicitConversion: true, // convert query strings to the right type
      },
    }),
  );

  // Optional: global API prefix
  app.setGlobalPrefix('api/v1');

  await app.listen(3000);
  console.log(`Application running on: ${await app.getUrl()}`);
}
bootstrap();
```

### Nested DTOs with ValidateNested

```typescript
import { ValidateNested, IsArray } from 'class-validator';
import { Type } from 'class-transformer';

class AddressDto {
  @IsString() street: string;
  @IsString() city: string;
}

export class CreateUserDto {
  @IsString() name: string;

  @ValidateNested()
  @Type(() => AddressDto)  // class-transformer needs this to know the class
  address: AddressDto;

  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => AddressDto)
  addresses: AddressDto[];
}
```

### Common class-validator decorators

| Decorator | Description |
|-----------|-------------|
| `@IsString()` | Must be a string |
| `@IsNumber()` | Must be a number |
| `@IsInt()` | Must be an integer |
| `@IsBoolean()` | Must be boolean |
| `@IsEmail()` | Valid email format |
| `@IsUrl()` | Valid URL |
| `@IsUUID()` | Valid UUID |
| `@IsDate()` | Valid Date |
| `@IsEnum(MyEnum)` | Must be an enum value |
| `@IsNotEmpty()` | Not empty string/null |
| `@IsOptional()` | Skip validation if absent |
| `@IsArray()` | Must be an array |
| `@MinLength(n)` | String min length |
| `@MaxLength(n)` | String max length |
| `@Min(n)` / `@Max(n)` | Number range |
| `@Matches(/regex/)` | Regex match |
| `@ValidateNested()` | Recurse into nested object |

### Creating custom DTO validation

When the built-in decorators aren't enough, you can build your own. There are
three levels: a **custom constraint class**, an **inline custom decorator**, and
a **fully custom validation pipe** for advanced cases.

#### 1. Custom validator with `@ValidatorConstraint` (reusable class)

Use this for a rule you'll reuse across many DTOs (e.g. "must be a strong
password"). Implement `ValidatorConstraintInterface`, then expose it as a
decorator with `registerDecorator`.

```typescript
import {
  registerDecorator,
  ValidationOptions,
  ValidatorConstraint,
  ValidatorConstraintInterface,
  ValidationArguments,
} from 'class-validator';

// The constraint holds the actual rule. `@ValidatorConstraint` registers it.
@ValidatorConstraint({ name: 'isStrongPassword', async: false })
export class IsStrongPasswordConstraint
  implements ValidatorConstraintInterface
{
  // Return true when valid, false when invalid.
  validate(value: string): boolean {
    if (typeof value !== 'string') return false;
    // At least 8 chars, one uppercase, one lowercase, one digit, one symbol.
    return /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^A-Za-z0-9]).{8,}$/.test(value);
  }

  // The message shown when validation fails. `args.property` is the field name.
  defaultMessage(args: ValidationArguments): string {
    return `${args.property} must be 8+ chars with upper, lower, number, and symbol`;
  }
}

// A clean decorator wrapper so DTOs can just use `@IsStrongPassword()`.
export function IsStrongPassword(validationOptions?: ValidationOptions) {
  return function (object: object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName,
      options: validationOptions,
      constraints: [],
      validator: IsStrongPasswordConstraint,
    });
  };
}
```

```typescript
// Use it in a DTO exactly like a built-in decorator:
export class RegisterDto {
  @IsEmail()
  email: string;

  @IsStrongPassword() // ← your custom rule
  password: string;
}
```

#### 2. Cross-field validation (compare two properties)

A common need: "confirmPassword must equal password". The validator receives the
whole object through `args.object`, so you can compare sibling fields. Pass the
other field name via `constraints`.

```typescript
import {
  registerDecorator,
  ValidationOptions,
  ValidationArguments,
} from 'class-validator';

// `property` is the name of the field to match against.
export function Match(property: string, validationOptions?: ValidationOptions) {
  return function (object: object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName,
      options: validationOptions,
      constraints: [property], // stored, read back in validate()
      validator: {
        validate(value: unknown, args: ValidationArguments) {
          const [relatedProperty] = args.constraints as string[];
          // Read the sibling field's value off the object being validated.
          const relatedValue = (args.object as Record<string, unknown>)[
            relatedProperty
          ];
          return value === relatedValue;
        },
        defaultMessage(args: ValidationArguments) {
          const [relatedProperty] = args.constraints as string[];
          return `${args.property} must match ${relatedProperty}`;
        },
      },
    });
  };
}
```

```typescript
export class ChangePasswordDto {
  @IsStrongPassword()
  password: string;

  @Match('password', { message: 'Passwords do not match' })
  confirmPassword: string;
}
```

#### 3. Async custom validation (e.g. check the database)

Set `async: true` and return a `Promise<boolean>`. Useful for "email already
taken" checks. The constraint can use DI if you register it as a provider.

```typescript
import { Injectable } from '@nestjs/common';
import {
  ValidatorConstraint,
  ValidatorConstraintInterface,
  ValidationArguments,
} from 'class-validator';
import { UsersService } from './users.service';

@ValidatorConstraint({ name: 'isEmailUnique', async: true })
@Injectable() // allows constructor injection (see "useContainer" note below)
export class IsEmailUniqueConstraint implements ValidatorConstraintInterface {
  constructor(private readonly usersService: UsersService) {}

  async validate(email: string): Promise<boolean> {
    const existing = await this.usersService.findByEmail(email);
    return !existing; // valid only if no user already has this email
  }

  defaultMessage(args: ValidationArguments): string {
    return `${args.property} is already registered`;
  }
}
```

> **⚡ DI note:** for a custom validator constraint to receive injected services,
> call `useContainer(app.select(AppModule), { fallbackOnErrors: true })` from
> `class-validator` in your `main.ts` (after creating the app). Without it,
> `class-validator` instantiates the constraint itself and dependencies are
> `undefined`.

```typescript
// main.ts
import { useContainer } from 'class-validator';
import { AppModule } from './app.module';

const app = await NestFactory.create(AppModule);
useContainer(app.select(AppModule), { fallbackOnErrors: true });
```

#### 4. A fully custom validation Pipe (advanced / non-DTO cases)

When you need logic outside the decorator model — transforming + validating raw
input — write a class implementing `PipeTransform`. (For most DTO validation,
prefer the built-in `ValidationPipe` above; reach for this only for special
cases like validating a single route param.)

```typescript
import {
  PipeTransform,
  Injectable,
  ArgumentMetadata,
  BadRequestException,
} from '@nestjs/common';

// Validates that a route param is a positive integer, and converts it.
@Injectable()
export class PositiveIntPipe implements PipeTransform<string, number> {
  transform(value: string, _metadata: ArgumentMetadata): number {
    const parsed = Number(value);
    if (!Number.isInteger(parsed) || parsed <= 0) {
      throw new BadRequestException('Parameter must be a positive integer');
    }
    return parsed; // the controller receives a clean number
  }
}

// Usage: @Get(':id') findOne(@Param('id', PositiveIntPipe) id: number) { ... }
```

> **Custom error messages:** every built-in decorator also accepts a `message`
> option, e.g. `@MinLength(8, { message: 'Password too short' })` or a function
> that builds the message from the validation arguments:
>
> ```typescript
> @IsString({ message: (args) => args.property + ' must be text' })
> ```

---

## 8. Pipes, Guards, Interceptors, Middleware & Exception Filters

These are the five **enhancement layers** around your controller handlers.

---

### Middleware

Runs **before** the route is matched. Equivalent to Express middleware. Good for logging, request parsing, CORS, rate limiting.

```typescript
// common/middleware/logger.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const { method, originalUrl } = req;
    const start = Date.now();

    res.on('finish', () => {
      const elapsed = Date.now() - start;
      console.log(`${method} ${originalUrl} ${res.statusCode} +${elapsed}ms`);
    });

    next(); // MUST call next() or the request hangs
  }
}
```

Apply in the module:

```typescript
// app.module.ts
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';

@Module({ /* ... */ })
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('*');  // or: { path: 'users', method: RequestMethod.GET }
  }
}
```

---

### Guards

Run **after middleware, before interceptors**. Answer one question: **should this request proceed?** Return `true` to allow, `false` to deny (403).

```typescript
// common/guards/roles.guard.ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  ForbiddenException,
} from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    // Read metadata set by @Roles() decorator
    const requiredRoles = this.reflector.getAllAndOverride<string[]>('roles', [
      context.getHandler(),   // method-level metadata first
      context.getClass(),     // then class-level
    ]);

    if (!requiredRoles) return true; // no roles required — allow

    const { user } = context.switchToHttp().getRequest();
    const hasRole = requiredRoles.some((role) => user?.roles?.includes(role));

    if (!hasRole) {
      throw new ForbiddenException('Insufficient permissions');
    }
    return true;
  }
}
```

Custom `@Roles()` decorator (used with the guard above):

```typescript
// common/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);

// Usage on controller:
// @Roles('admin')
// @UseGuards(JwtAuthGuard, RolesGuard)
// @Delete(':id')
// remove(...) {}
```

Apply guards globally or per-route:

```typescript
// Per route
@UseGuards(AuthGuard)

// Per controller
@Controller('users')
@UseGuards(AuthGuard)
export class UsersController {}

// Globally in main.ts
app.useGlobalGuards(new AuthGuard());

// Globally via DI (preferred — allows injection)
// In app.module.ts providers:
{ provide: APP_GUARD, useClass: JwtAuthGuard }
```

---

### Interceptors

Wrap the handler execution — run code **before and after**. Use for: response transformation, logging, caching, timeout, serialization.

```typescript
// common/interceptors/transform.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

// Wraps ALL responses in { data: ..., statusCode: ... }
@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, { data: T }> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<{ data: T }> {
    return next.handle().pipe(
      map((data) => ({
        data,
        statusCode: context.switchToHttp().getResponse().statusCode,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}
```

Timeout interceptor:

```typescript
// common/interceptors/timeout.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000), // 5 second timeout
      catchError((err) => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  }
}
```

Register globally:

```typescript
// main.ts
app.useGlobalInterceptors(new TransformInterceptor(), new TimeoutInterceptor());

// Or via DI in a module:
{ provide: APP_INTERCEPTOR, useClass: TransformInterceptor }
```

---

### Pipes

Transform and validate input **before it reaches the handler**. Built-in pipes cover the common cases:

| Built-in Pipe | Purpose |
|---------------|---------|
| `ValidationPipe` | Validate DTOs with class-validator |
| `ParseIntPipe` | String → integer (throws 400 if not numeric) |
| `ParseFloatPipe` | String → float |
| `ParseBoolPipe` | String → boolean |
| `ParseArrayPipe` | String/JSON → array |
| `ParseUUIDPipe` | Validates UUID format |
| `ParseEnumPipe` | Validates enum membership |
| `DefaultValuePipe` | Provide a default if value is undefined |

```typescript
// Using multiple built-in pipes
@Get()
findAll(
  @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
  @Query('limit', new DefaultValuePipe(10), ParseIntPipe) limit: number,
) {
  return this.usersService.findAll({ page, limit });
}
```

Custom pipe:

```typescript
// common/pipes/trim.pipe.ts
import { Injectable, PipeTransform, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class TrimPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    if (typeof value === 'string') {
      return value.trim();
    }
    if (typeof value === 'object' && value !== null) {
      // Trim all string properties
      Object.keys(value).forEach((key) => {
        if (typeof value[key] === 'string') value[key] = value[key].trim();
      });
    }
    return value;
  }
}
```

---

### Exception Filters

Catch **any exception** thrown anywhere in the request pipeline and return a formatted error response.

```typescript
// common/filters/http-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException) // catches only HttpException and its subclasses
export class HttpExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(HttpExceptionFilter.name);

  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();
    const exceptionResponse = exception.getResponse();

    const errorBody = {
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      method: request.method,
      message:
        typeof exceptionResponse === 'object'
          ? (exceptionResponse as any).message
          : exceptionResponse,
    };

    this.logger.error(
      `${request.method} ${request.url} ${status}`,
      JSON.stringify(errorBody),
    );

    response.status(status).json(errorBody);
  }
}
```

Catch-all filter (catches everything):

```typescript
@Catch() // no argument = catch all exceptions
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    response.status(status).json({
      statusCode: status,
      message: exception instanceof Error ? exception.message : 'Internal server error',
    });
  }
}
```

Register globally:

```typescript
// main.ts
app.useGlobalFilters(new HttpExceptionFilter());

// Or via DI:
{ provide: APP_FILTER, useClass: HttpExceptionFilter }
```

---

## 9. Configuration (@nestjs/config)

Manage environment variables with type safety and validation.

```bash
npm install @nestjs/config
npm install @hapi/joi   # or: npm install joi  (for schema validation of env vars)
```

### Setup

```typescript
// config/configuration.ts — typed config factory
export default () => ({
  port: parseInt(process.env.PORT ?? '3000', 10),
  database: {
    url: process.env.DATABASE_URL,
    ssl: process.env.DATABASE_SSL === 'true',
  },
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRES_IN ?? '7d',
  },
  redis: {
    host: process.env.REDIS_HOST ?? 'localhost',
    port: parseInt(process.env.REDIS_PORT ?? '6379', 10),
  },
});
```

```typescript
// app.module.ts
import { ConfigModule } from '@nestjs/config';
import * as Joi from 'joi';
import configuration from './config/configuration';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,          // no need to import ConfigModule in every module
      load: [configuration],   // load your typed config factory
      envFilePath: ['.env.local', '.env'], // load multiple env files (first wins)
      validationSchema: Joi.object({
        NODE_ENV: Joi.string().valid('development', 'production', 'test').default('development'),
        PORT: Joi.number().default(3000),
        DATABASE_URL: Joi.string().required(),
        JWT_SECRET: Joi.string().required().min(32),
      }),
      validationOptions: {
        abortEarly: false, // show ALL validation errors at startup, not just first
      },
    }),
  ],
})
export class AppModule {}
```

### Using ConfigService

```typescript
// any.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class AnyService {
  constructor(private configService: ConfigService) {}

  example() {
    // Typed access with default value
    const port = this.configService.get<number>('port');
    const dbUrl = this.configService.get<string>('database.url');
    const jwtSecret = this.configService.getOrThrow<string>('jwt.secret'); // throws if missing
  }
}
```

### .env file example

```bash
# .env
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
JWT_SECRET=super-long-secret-key-at-least-32-chars-long
JWT_EXPIRES_IN=7d
REDIS_HOST=localhost
REDIS_PORT=6379
```

> **Security tip:** Never commit `.env` to git. Add it to `.gitignore`. Commit a `.env.example` with placeholder values.

---

## 10. Database Integration — TypeORM, Prisma & Mongoose

### TypeORM

The most common ORM in the NestJS ecosystem. Works with PostgreSQL, MySQL, SQLite, and more.

```bash
npm install @nestjs/typeorm typeorm pg   # pg = postgres driver
```

```typescript
// app.module.ts (TypeORM setup)
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        type: 'postgres',
        url: configService.getOrThrow('DATABASE_URL'),
        autoLoadEntities: true,  // picks up entities registered with forFeature()
        synchronize: configService.get('NODE_ENV') === 'development', // auto-migrate (dev only!)
        logging: configService.get('NODE_ENV') === 'development',
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

#### Define an entity

```typescript
// users/entities/user.entity.ts
import {
  Entity,
  Column,
  PrimaryGeneratedColumn,
  CreateDateColumn,
  UpdateDateColumn,
  OneToMany,
  Index,
  BeforeInsert,
} from 'typeorm';
import * as bcrypt from 'bcrypt';

@Entity('users')  // table name
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ length: 50 })
  name: string;

  @Index({ unique: true })
  @Column({ unique: true })
  email: string;

  @Column({ select: false }) // never returned in queries by default
  password: string;

  @Column({ type: 'enum', enum: ['admin', 'user'], default: 'user' })
  role: string;

  @Column({ default: true })
  isActive: boolean;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  // Hash password before inserting
  @BeforeInsert()
  async hashPassword() {
    this.password = await bcrypt.hash(this.password, 12);
  }
}
```

#### Repository pattern in the service

```typescript
// users/users.module.ts — register the entity
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './entities/user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])], // registers UserRepository
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

```typescript
// users/users.service.ts
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './entities/user.entity';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}

  findAll(): Promise<User[]> {
    return this.userRepository.find({
      where: { isActive: true },
      order: { createdAt: 'DESC' },
    });
  }

  findOne(id: string): Promise<User | null> {
    return this.userRepository.findOne({ where: { id } });
  }

  findByEmail(email: string): Promise<User | null> {
    return this.userRepository
      .createQueryBuilder('user')
      .addSelect('user.password') // override select:false for this query
      .where('user.email = :email', { email })
      .getOne();
  }

  async create(dto: CreateUserDto): Promise<User> {
    const user = this.userRepository.create(dto); // creates instance, runs @BeforeInsert
    return this.userRepository.save(user);
  }

  async update(id: string, dto: UpdateUserDto): Promise<User> {
    await this.userRepository.update(id, dto);
    return this.findOne(id);
  }

  async remove(id: string): Promise<void> {
    await this.userRepository.delete(id);
  }

  // Transaction example
  async transferCredits(fromId: string, toId: string, amount: number) {
    return this.userRepository.manager.transaction(async (manager) => {
      const from = await manager.findOne(User, { where: { id: fromId } });
      const to = await manager.findOne(User, { where: { id: toId } });
      // ... update both records within the same transaction
    });
  }
}
```

---

### Prisma

Prisma is a **schema-first** ORM with an auto-generated, type-safe client. Many developers prefer it over TypeORM for its superior DX.

```bash
npm install prisma @prisma/client
npx prisma init   # creates prisma/schema.prisma and .env
```

#### schema.prisma

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  name      String
  email     String   @unique
  password  String
  role      Role     @default(USER)
  posts     Post[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Post {
  id        String   @id @default(uuid())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  String
  createdAt DateTime @default(now())
}

enum Role {
  USER
  ADMIN
  MODERATOR
}
```

```bash
# Generate the client + run migrations
npx prisma migrate dev --name init   # dev: creates migration files
npx prisma generate                   # regenerate client after schema changes
npx prisma studio                     # open GUI to browse data
```

#### Prisma service (NestJS wrapper)

```typescript
// database/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

```typescript
// database/database.module.ts
import { Module, Global } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class DatabaseModule {}
```

#### Using Prisma in a service

```typescript
// users/users.service.ts (Prisma version)
import { Injectable, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../database/prisma.service';
import { CreateUserDto } from './dto/create-user.dto';
import * as bcrypt from 'bcrypt';

@Injectable()
export class UsersService {
  constructor(private readonly prisma: PrismaService) {}

  findAll() {
    return this.prisma.user.findMany({
      where: { },
      orderBy: { createdAt: 'desc' },
      select: {                        // explicitly exclude password
        id: true,
        name: true,
        email: true,
        role: true,
        createdAt: true,
      },
    });
  }

  async findOne(id: string) {
    const user = await this.prisma.user.findUnique({ where: { id } });
    if (!user) throw new NotFoundException(`User ${id} not found`);
    return user;
  }

  async create(dto: CreateUserDto) {
    const hashed = await bcrypt.hash(dto.password, 12);
    return this.prisma.user.create({
      data: { ...dto, password: hashed },
    });
  }

  update(id: string, dto: Partial<CreateUserDto>) {
    return this.prisma.user.update({ where: { id }, data: dto });
  }

  remove(id: string) {
    return this.prisma.user.delete({ where: { id } });
  }

  // Relations — include related data
  findWithPosts(id: string) {
    return this.prisma.user.findUnique({
      where: { id },
      include: { posts: { where: { published: true } } },
    });
  }

  // Transaction
  async createUserWithPost(userDto: CreateUserDto, postTitle: string) {
    return this.prisma.$transaction(async (tx) => {
      const user = await tx.user.create({ data: userDto });
      const post = await tx.post.create({
        data: { title: postTitle, authorId: user.id },
      });
      return { user, post };
    });
  }
}
```

---

### Mongoose (MongoDB)

```bash
npm install @nestjs/mongoose mongoose
```

```typescript
// app.module.ts
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot(process.env.MONGODB_URI),
  ],
})
export class AppModule {}
```

```typescript
// users/schemas/user.schema.ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

export type UserDocument = User & Document;

@Schema({ timestamps: true })
export class User {
  @Prop({ required: true })
  name: string;

  @Prop({ required: true, unique: true, lowercase: true })
  email: string;

  @Prop({ required: true, select: false })
  password: string;
}

export const UserSchema = SchemaFactory.createForClass(User);
```

```typescript
// users/users.module.ts
MongooseModule.forFeature([{ name: User.name, schema: UserSchema }])

// users/users.service.ts
constructor(@InjectModel(User.name) private userModel: Model<UserDocument>) {}
```

---

## 11. Authentication & Authorization (Passport + JWT)

### Install

```bash
npm install @nestjs/passport passport passport-jwt
npm install @nestjs/jwt
npm install --save-dev @types/passport-jwt
npm install bcrypt
npm install --save-dev @types/bcrypt
```

### JWT Strategy

```typescript
// auth/strategies/jwt.strategy.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';
import { UsersService } from '../../users/users.service';

// Payload stored in the JWT token
export interface JwtPayload {
  sub: string;  // user id (standard JWT claim)
  email: string;
  role: string;
}

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(
    configService: ConfigService,
    private usersService: UsersService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(), // Authorization: Bearer <token>
      ignoreExpiration: false,
      secretOrKey: configService.getOrThrow('jwt.secret'),
    });
  }

  // Called after token is verified — return value is attached to request.user
  async validate(payload: JwtPayload) {
    const user = await this.usersService.findOne(payload.sub);
    if (!user) throw new UnauthorizedException();
    return user; // this becomes req.user
  }
}
```

### Auth Service

```typescript
// auth/auth.service.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { UsersService } from '../users/users.service';
import * as bcrypt from 'bcrypt';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService,
  ) {}

  async validateUser(email: string, password: string) {
    // findByEmail must select password (it's excluded by default)
    const user = await this.usersService.findByEmail(email);
    if (!user) throw new UnauthorizedException('Invalid credentials');

    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) throw new UnauthorizedException('Invalid credentials');

    return user;
  }

  async login(email: string, password: string) {
    const user = await this.validateUser(email, password);

    const payload: JwtPayload = {
      sub: user.id,
      email: user.email,
      role: user.role,
    };

    return {
      accessToken: this.jwtService.sign(payload),
      user: { id: user.id, email: user.email, name: user.name, role: user.role },
    };
  }

  async register(dto: CreateUserDto) {
    return this.usersService.create(dto);
  }
}
```

### Auth Module

```typescript
// auth/auth.module.ts
import { Module } from '@nestjs/common';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { JwtStrategy } from './strategies/jwt.strategy';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [
    UsersModule,
    PassportModule.register({ defaultStrategy: 'jwt' }),
    JwtModule.registerAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        secret: configService.getOrThrow('jwt.secret'),
        signOptions: {
          expiresIn: configService.get('jwt.expiresIn', '7d'),
        },
      }),
      inject: [ConfigService],
    }),
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy],
  exports: [AuthService, JwtModule],
})
export class AuthModule {}
```

### Auth Controller

```typescript
// auth/auth.controller.ts
import { Controller, Post, Body, HttpCode, HttpStatus } from '@nestjs/common';
import { AuthService } from './auth.service';

class LoginDto {
  email: string;
  password: string;
}

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post('register')
  register(@Body() dto: CreateUserDto) {
    return this.authService.register(dto);
  }

  @Post('login')
  @HttpCode(HttpStatus.OK) // POST returns 201 by default; override to 200 for login
  login(@Body() dto: LoginDto) {
    return this.authService.login(dto.email, dto.password);
  }
}
```

### JWT Auth Guard

```typescript
// auth/guards/jwt-auth.guard.ts
import { Injectable, ExecutionContext } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { Reflector } from '@nestjs/core';

// Marker for public routes
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    // Check if route is marked @Public()
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;

    return super.canActivate(context);
  }
}
```

Register globally so all routes require auth by default:

```typescript
// app.module.ts — providers array
{ provide: APP_GUARD, useClass: JwtAuthGuard }
```

Accessing current user in a controller:

```typescript
// common/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user; // set by JwtStrategy.validate()
  },
);

// Usage in controller:
@Get('me')
getProfile(@CurrentUser() user: User) {
  return user;
}
```

---

## 12. OpenAPI / Swagger Docs

Auto-generate interactive API docs from your decorators.

```bash
npm install @nestjs/swagger
```

```typescript
// main.ts
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));

  // Swagger setup
  const config = new DocumentBuilder()
    .setTitle('My API')
    .setDescription('The my-api API description')
    .setVersion('1.0')
    .addBearerAuth()      // adds the "Authorize" button for JWT
    .addTag('users', 'User management')
    .addTag('auth', 'Authentication')
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api/docs', app, document, {
    swaggerOptions: { persistAuthorization: true }, // remembers JWT between page refreshes
  });

  await app.listen(3000);
  console.log(`Swagger UI: http://localhost:3000/api/docs`);
}
```

### Decorating your DTOs and controllers

```typescript
// dto/create-user.dto.ts
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class CreateUserDto {
  @ApiProperty({ example: 'Alice Johnson', description: 'Full name' })
  @IsString()
  name: string;

  @ApiProperty({ example: 'alice@example.com' })
  @IsEmail()
  email: string;

  @ApiProperty({ example: 'Str0ng!Pass', minLength: 8 })
  @IsString()
  @MinLength(8)
  password: string;

  @ApiPropertyOptional({ enum: UserRole, default: UserRole.USER })
  @IsEnum(UserRole)
  @IsOptional()
  role?: UserRole;
}
```

```typescript
// users/users.controller.ts
import {
  ApiTags,
  ApiOperation,
  ApiResponse,
  ApiBearerAuth,
  ApiParam,
  ApiQuery,
} from '@nestjs/swagger';

@ApiTags('users')         // groups endpoints in Swagger UI
@ApiBearerAuth()          // marks all endpoints as requiring JWT
@Controller('users')
export class UsersController {
  @ApiOperation({ summary: 'Get all users', description: 'Returns paginated user list' })
  @ApiQuery({ name: 'page', required: false, type: Number })
  @ApiResponse({ status: 200, description: 'List of users', type: [User] })
  @Get()
  findAll() { /* ... */ }

  @ApiOperation({ summary: 'Get user by ID' })
  @ApiParam({ name: 'id', type: String, description: 'User UUID' })
  @ApiResponse({ status: 200, description: 'User found', type: User })
  @ApiResponse({ status: 404, description: 'User not found' })
  @Get(':id')
  findOne(@Param('id') id: string) { /* ... */ }
}
```

> **⚡ Version note:** In NestJS 11 with `@nestjs/swagger` v8+, the plugin can auto-extract DTO descriptions from JSDoc comments. Add it to `nest-cli.json`:
> ```json
> { "compilerOptions": { "plugins": ["@nestjs/swagger"] } }
> ```
> This removes the need to manually add `@ApiProperty()` to every DTO field.

---

## 13. WebSockets & Gateways

```bash
npm install @nestjs/websockets @nestjs/platform-socket.io socket.io
```

### Basic Gateway

```typescript
// events/events.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  MessageBody,
  ConnectedSocket,
  OnGatewayConnection,
  OnGatewayDisconnect,
  OnGatewayInit,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { Logger } from '@nestjs/common';

@WebSocketGateway({
  namespace: '/chat',       // optional — default is '/'
  cors: { origin: '*' },   // configure in production
})
export class EventsGateway
  implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect
{
  @WebSocketServer()
  server: Server; // the socket.io Server instance

  private readonly logger = new Logger(EventsGateway.name);

  afterInit() {
    this.logger.log('WebSocket Gateway initialized');
  }

  handleConnection(client: Socket) {
    this.logger.log(`Client connected: ${client.id}`);
  }

  handleDisconnect(client: Socket) {
    this.logger.log(`Client disconnected: ${client.id}`);
  }

  // Handle event named 'message'
  @SubscribeMessage('message')
  handleMessage(
    @MessageBody() data: { room: string; text: string },
    @ConnectedSocket() client: Socket,
  ) {
    // Broadcast to everyone in the room (except sender)
    client.to(data.room).emit('message', {
      text: data.text,
      sender: client.id,
      timestamp: new Date().toISOString(),
    });

    // Return value is sent back to the sender as acknowledgement
    return { event: 'message', data: 'Message sent' };
  }

  @SubscribeMessage('joinRoom')
  handleJoinRoom(
    @MessageBody() room: string,
    @ConnectedSocket() client: Socket,
  ) {
    client.join(room);
    client.to(room).emit('userJoined', { userId: client.id });
    return { joined: room };
  }

  // Push from server to all clients (call from a service)
  broadcastToAll(event: string, data: any) {
    this.server.emit(event, data);
  }

  // Push to a specific room
  emitToRoom(room: string, event: string, data: any) {
    this.server.to(room).emit(event, data);
  }
}
```

```typescript
// events/events.module.ts
@Module({
  providers: [EventsGateway],
  exports: [EventsGateway],
})
export class EventsModule {}
```

### Client-side connection (JavaScript)

```javascript
// Browser or Node.js client
import { io } from 'socket.io-client';

const socket = io('http://localhost:3000/chat');

socket.on('connect', () => {
  console.log('Connected:', socket.id);
  socket.emit('joinRoom', 'general');
});

socket.emit('message', { room: 'general', text: 'Hello everyone!' });
socket.on('message', (msg) => console.log('Received:', msg));
```

---

## 14. File Uploads

```bash
npm install @nestjs/platform-express
# Multer is bundled with @nestjs/platform-express — no separate install needed
npm install --save-dev @types/multer
```

### Single file upload

```typescript
// upload/upload.controller.ts
import {
  Controller,
  Post,
  UploadedFile,
  UseInterceptors,
  BadRequestException,
  Get,
  Param,
  Res,
} from '@nestjs/common';
import { FileInterceptor, FilesInterceptor } from '@nestjs/platform-express';
import { diskStorage } from 'multer';
import { extname, join } from 'path';
import { Response } from 'express';
import { Express } from 'express';

// Helper: generate a unique filename
const editFileName = (req: any, file: Express.Multer.File, callback: Function) => {
  const uniqueSuffix = `${Date.now()}-${Math.round(Math.random() * 1e9)}`;
  const ext = extname(file.originalname);
  callback(null, `${file.fieldname}-${uniqueSuffix}${ext}`);
};

// Helper: filter by file type
const imageFileFilter = (req: any, file: Express.Multer.File, callback: Function) => {
  if (!file.mimetype.match(/^image\/(jpg|jpeg|png|gif|webp)$/)) {
    return callback(new BadRequestException('Only image files are allowed'), false);
  }
  callback(null, true);
};

@Controller('upload')
export class UploadController {
  // Single file field named 'avatar'
  @Post('avatar')
  @UseInterceptors(
    FileInterceptor('avatar', {
      storage: diskStorage({
        destination: './uploads/avatars',
        filename: editFileName,
      }),
      fileFilter: imageFileFilter,
      limits: { fileSize: 5 * 1024 * 1024 }, // 5 MB max
    }),
  )
  uploadAvatar(@UploadedFile() file: Express.Multer.File) {
    if (!file) throw new BadRequestException('File is required');
    return {
      filename: file.filename,
      originalName: file.originalname,
      size: file.size,
      url: `/upload/files/${file.filename}`,
    };
  }

  // Multiple files (up to 10) from field 'photos'
  @Post('photos')
  @UseInterceptors(FilesInterceptor('photos', 10, { /* same options */ }))
  uploadPhotos(@UploadedFiles() files: Express.Multer.File[]) {
    return files.map((f) => ({ filename: f.filename, size: f.size }));
  }

  // Serve uploaded files
  @Get('files/:filename')
  serveFile(@Param('filename') filename: string, @Res() res: Response) {
    res.sendFile(join(process.cwd(), 'uploads/avatars', filename));
  }
}
```

### In-memory storage (for processing, not saving)

```typescript
@UseInterceptors(FileInterceptor('file', { storage: memoryStorage() }))
async processFile(@UploadedFile() file: Express.Multer.File) {
  // file.buffer contains the raw bytes
  const text = file.buffer.toString('utf-8');
  // parse CSV, resize image, etc.
}
```

### Serve static files globally

```typescript
// main.ts
import { ServeStaticModule } from '@nestjs/serve-static';
import { join } from 'path';

// In app.module.ts imports:
ServeStaticModule.forRoot({
  rootPath: join(__dirname, '..', 'uploads'),
  serveRoot: '/uploads', // accessible at http://localhost:3000/uploads/
})
```

---

## 15. Caching, Task Scheduling & Queues

### Caching

```bash
npm install @nestjs/cache-manager cache-manager
npm install cache-manager-redis-store  # for Redis backend
```

```typescript
// app.module.ts
import { CacheModule } from '@nestjs/cache-manager';

@Module({
  imports: [
    CacheModule.register({
      isGlobal: true,
      ttl: 60,   // seconds
      max: 100,  // max items in memory cache
    }),
    // For Redis:
    // CacheModule.registerAsync({
    //   useFactory: () => ({ store: redisStore, host: 'localhost', port: 6379, ttl: 60 })
    // })
  ],
})
export class AppModule {}
```

```typescript
// In a controller — auto-cache GET responses
import { CacheInterceptor, CacheKey, CacheTTL } from '@nestjs/cache-manager';

@UseInterceptors(CacheInterceptor)
@CacheKey('all-users')
@CacheTTL(30)   // override TTL to 30s
@Get()
findAll() {
  return this.usersService.findAll();
}
```

```typescript
// Manual cache control in a service
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class ProductsService {
  constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}

  async findAll() {
    const cached = await this.cacheManager.get<Product[]>('products');
    if (cached) return cached;

    const products = await this.db.product.findMany();
    await this.cacheManager.set('products', products, 60); // 60s TTL
    return products;
  }

  async invalidateCache() {
    await this.cacheManager.del('products');
  }
}
```

---

### Task Scheduling

```bash
npm install @nestjs/schedule
npm install --save-dev @types/cron
```

```typescript
// app.module.ts
import { ScheduleModule } from '@nestjs/schedule';
@Module({ imports: [ScheduleModule.forRoot()] })
export class AppModule {}
```

```typescript
// tasks/tasks.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron, Interval, Timeout, CronExpression } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  // Cron syntax — every day at midnight
  @Cron(CronExpression.EVERY_DAY_AT_MIDNIGHT)
  cleanupExpiredSessions() {
    this.logger.log('Cleaning up expired sessions...');
  }

  // Custom cron — every Monday at 9am
  @Cron('0 9 * * 1')
  sendWeeklyReport() {
    this.logger.log('Sending weekly report...');
  }

  // Every 30 seconds
  @Interval(30000)
  pollExternalApi() {
    this.logger.log('Polling external API...');
  }

  // Run once after 5 seconds of startup
  @Timeout(5000)
  handleStartupTask() {
    this.logger.log('Startup task ran!');
  }
}
```

---

### Queues with BullMQ

For heavy/async work: email sending, image processing, report generation.

```bash
npm install @nestjs/bullmq bullmq ioredis
```

```typescript
// app.module.ts
import { BullModule } from '@nestjs/bullmq';

@Module({
  imports: [
    BullModule.forRoot({ connection: { host: 'localhost', port: 6379 } }),
    BullModule.registerQueue({ name: 'emails' }),
  ],
})
export class AppModule {}
```

```typescript
// email/email.producer.ts — add jobs to the queue
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';

@Injectable()
export class EmailProducer {
  constructor(@InjectQueue('emails') private emailQueue: Queue) {}

  async sendWelcomeEmail(userId: string, email: string) {
    await this.emailQueue.add('welcome', { userId, email }, {
      attempts: 3,        // retry up to 3 times
      backoff: { type: 'exponential', delay: 1000 },
      removeOnComplete: true,
    });
  }
}
```

```typescript
// email/email.consumer.ts — process jobs
import { Processor, WorkerHost } from '@nestjs/bullmq';
import { Job } from 'bullmq';

@Processor('emails')
export class EmailConsumer extends WorkerHost {
  async process(job: Job) {
    switch (job.name) {
      case 'welcome':
        const { userId, email } = job.data;
        // send email via nodemailer / SendGrid / etc.
        console.log(`Sending welcome email to ${email}`);
        break;
    }
  }
}
```

---

## 16. Microservices Overview

NestJS supports multiple transport layers for microservices out of the box.

```bash
npm install @nestjs/microservices
# For Redis transport:
npm install ioredis
# For RabbitMQ:
npm install amqplib amqp-connection-manager
# For Kafka:
npm install kafkajs
```

### Microservice app bootstrap

```typescript
// microservice main.ts
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.REDIS,
      options: { host: 'localhost', port: 6379 },
    },
  );
  await app.listen();
}
```

### Message patterns in a microservice controller

```typescript
import { Controller } from '@nestjs/common';
import { MessagePattern, EventPattern, Payload, Ctx, RedisContext } from '@nestjs/microservices';

@Controller()
export class OrdersController {
  // Request-response pattern (client awaits reply)
  @MessagePattern({ cmd: 'get_orders' })
  getOrders(@Payload() data: { userId: string }) {
    return this.ordersService.findByUser(data.userId);
  }

  // Fire-and-forget event (no reply)
  @EventPattern('order_created')
  handleOrderCreated(@Payload() data: any, @Ctx() context: RedisContext) {
    console.log(`Order created event received:`, data);
  }
}
```

### Client sending messages to a microservice

```typescript
import { ClientsModule, Transport } from '@nestjs/microservices';

// In app.module.ts imports:
ClientsModule.register([
  { name: 'ORDERS_SERVICE', transport: Transport.REDIS, options: { host: 'localhost', port: 6379 } },
])

// In a service:
constructor(@Inject('ORDERS_SERVICE') private client: ClientProxy) {}

getOrders(userId: string) {
  return this.client.send({ cmd: 'get_orders' }, { userId }); // returns Observable
}

notifyOrderCreated(order: any) {
  this.client.emit('order_created', order); // fire and forget
}
```

### Available transports

| Transport | Use case |
|-----------|---------|
| `TCP` | Internal network, simple |
| `Redis` | Pub/sub, simple queuing |
| `NATS` | High-throughput messaging |
| `RabbitMQ` | Reliable message delivery, enterprise |
| `Kafka` | High-throughput event streaming |
| `gRPC` | High-performance, type-safe RPC |
| `MQTT` | IoT, lightweight |

---

## 17. Testing — Unit & E2E

NestJS ships with Jest pre-configured. `nest g resource` generates test files automatically.

### Unit testing a service

```typescript
// users/users.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UsersService } from './users.service';
import { getRepositoryToken } from '@nestjs/typeorm';
import { User } from './entities/user.entity';
import { Repository } from 'typeorm';
import { NotFoundException } from '@nestjs/common';

// Mock repository factory
const mockRepository = () => ({
  find: jest.fn(),
  findOne: jest.fn(),
  create: jest.fn(),
  save: jest.fn(),
  update: jest.fn(),
  delete: jest.fn(),
});

type MockRepository<T = any> = Partial<Record<keyof Repository<T>, jest.Mock>>;

describe('UsersService', () => {
  let service: UsersService;
  let userRepository: MockRepository<User>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: getRepositoryToken(User),
          useFactory: mockRepository,
        },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
    userRepository = module.get<MockRepository<User>>(getRepositoryToken(User));
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  describe('findOne', () => {
    it('should return a user when found', async () => {
      const user = { id: '1', name: 'Alice', email: 'alice@test.com' };
      userRepository.findOne.mockResolvedValue(user);

      const result = await service.findOne('1');
      expect(result).toEqual(user);
      expect(userRepository.findOne).toHaveBeenCalledWith({ where: { id: '1' } });
    });

    it('should throw NotFoundException when user not found', async () => {
      userRepository.findOne.mockResolvedValue(null);

      await expect(service.findOne('999')).rejects.toThrow(NotFoundException);
    });
  });

  describe('create', () => {
    it('should create and return a user', async () => {
      const dto = { name: 'Bob', email: 'bob@test.com', password: 'secret123' };
      const savedUser = { id: '2', ...dto };

      userRepository.create.mockReturnValue(savedUser);
      userRepository.save.mockResolvedValue(savedUser);

      const result = await service.create(dto as any);
      expect(result).toEqual(savedUser);
    });
  });
});
```

### Unit testing a controller

```typescript
// users/users.controller.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

const mockUsersService = {
  findAll: jest.fn(),
  findOne: jest.fn(),
  create: jest.fn(),
  update: jest.fn(),
  remove: jest.fn(),
};

describe('UsersController', () => {
  let controller: UsersController;
  let service: UsersService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [{ provide: UsersService, useValue: mockUsersService }],
    }).compile();

    controller = module.get<UsersController>(UsersController);
    service = module.get<UsersService>(UsersService);
  });

  it('should call service.findAll', async () => {
    const users = [{ id: '1', name: 'Alice' }];
    mockUsersService.findAll.mockResolvedValue(users);

    const result = await controller.findAll();
    expect(result).toEqual(users);
    expect(service.findAll).toHaveBeenCalled();
  });
});
```

### E2E Testing with Supertest

```typescript
// test/users.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';
import { DataSource } from 'typeorm';

describe('UsersController (e2e)', () => {
  let app: INestApplication;
  let dataSource: DataSource;
  let authToken: string;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule], // use real AppModule — spins up real DB (use test DB!)
    }).compile();

    app = moduleFixture.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
    await app.init();

    dataSource = moduleFixture.get(DataSource);

    // Login and get token for authenticated tests
    const loginRes = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ email: 'admin@test.com', password: 'Test1234!' });
    authToken = loginRes.body.accessToken;
  });

  afterAll(async () => {
    await dataSource.destroy();
    await app.close();
  });

  describe('GET /users', () => {
    it('should return 401 when not authenticated', () => {
      return request(app.getHttpServer())
        .get('/users')
        .expect(401);
    });

    it('should return user list when authenticated', () => {
      return request(app.getHttpServer())
        .get('/users')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200)
        .expect((res) => {
          expect(Array.isArray(res.body)).toBe(true);
        });
    });
  });

  describe('POST /users', () => {
    it('should create a user', () => {
      return request(app.getHttpServer())
        .post('/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ name: 'Test User', email: 'testuser@test.com', password: 'Pass1234!' })
        .expect(201)
        .expect((res) => {
          expect(res.body).toHaveProperty('id');
          expect(res.body.email).toBe('testuser@test.com');
          expect(res.body).not.toHaveProperty('password'); // never expose password
        });
    });

    it('should return 400 for invalid email', () => {
      return request(app.getHttpServer())
        .post('/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ name: 'Test', email: 'not-an-email', password: 'Pass1234!' })
        .expect(400);
    });
  });
});
```

### Jest config tips

```json
// package.json scripts
{
  "test": "jest",
  "test:watch": "jest --watch",
  "test:cov": "jest --coverage",
  "test:e2e": "jest --config ./test/jest-e2e.json"
}
```

```json
// test/jest-e2e.json
{
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": ".",
  "testEnvironment": "node",
  "testRegex": ".e2e-spec.ts$",
  "transform": { "^.+\\.(t|j)s$": "ts-jest" },
  "moduleNameMapper": { "^src/(.*)$": "<rootDir>/../src/$1" }
}
```

---

## 18. Tips, Tricks & Gotchas

### Architecture & Design

**Keep controllers thin.** Controllers should only: extract data from the request, call a service method, and return the result. Zero business logic.

**Keep services business-logic-only.** Avoid importing `Request`/`Response` in services — services should be reusable in microservices, queues, and WebSocket gateways, not just HTTP.

**One module per feature, not one module per layer.** A `UsersModule` is right. A `ServicesModule` that contains all services is wrong.

**Export only what other modules need.** Be conservative with `exports`. If you export a service, consider whether you should instead move the logic to a shared module.

---

### Common Gotchas

**Circular dependency.** When Module A imports Module B and Module B imports Module A, use `forwardRef`:

```typescript
// users.module.ts
imports: [forwardRef(() => AuthModule)]

// auth.module.ts
imports: [forwardRef(() => UsersModule)]

// In the service constructor:
constructor(@Inject(forwardRef(() => AuthService)) private authService: AuthService) {}
```

**`@Res()` breaks interceptors.** As mentioned in Section 5 — always use `{ passthrough: true }` when injecting `@Res()`.

**`synchronize: true` in TypeORM will destroy data.** Never use it in staging or production. Use migrations instead:
```bash
npx typeorm migration:generate src/migrations/AddUserTable -d src/data-source.ts
npx typeorm migration:run -d src/data-source.ts
```

**ValidationPipe without `transform: true` leaves DTOs as plain objects.** Class methods, `@BeforeInsert`, etc. won't work. Always set `transform: true`.

**`whitelist: true` silently strips extra fields.** If you also want a 400 error when extra fields are sent, add `forbidNonWhitelisted: true`. Choose based on whether your API needs to be strict.

**Global pipes/guards/interceptors registered in `main.ts` can't use DI.** They're instantiated before the DI container. Use the `APP_PIPE`, `APP_GUARD`, `APP_INTERCEPTOR` tokens in a module's `providers` array instead:
```typescript
{ provide: APP_PIPE, useClass: ValidationPipe }
```

**Request-scoped providers are a performance trap.** Every request creates new instances. Only use `Scope.REQUEST` when you genuinely need per-request state (e.g., tenant ID, request tracing).

**Don't return `password` from your user entity.** Add `select: false` to the TypeORM column, or use a class-transformer `@Exclude()` with `ClassSerializerInterceptor`:
```typescript
// On the entity property:
@Exclude() // requires ClassSerializerInterceptor to be applied
password: string;

// Globally in app.module.ts providers:
{ provide: APP_INTERCEPTOR, useClass: ClassSerializerInterceptor }
```

**`async/await` everywhere.** NestJS handles Promises and Observables — but mixing them carelessly causes bugs. If your service method is async, make the controller method async too.

**CORS isn't enabled by default.** Enable it:
```typescript
// main.ts
app.enableCors({
  origin: process.env.FRONTEND_URL || 'http://localhost:4200',
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  credentials: true,
});
```

**Helmet and rate limiting for production:**
```bash
npm install helmet @nestjs/throttler
```
```typescript
// main.ts
import helmet from 'helmet';
app.use(helmet());

// app.module.ts
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';
ThrottlerModule.forRoot([{ ttl: 60000, limit: 100 }]) // 100 req/min

// providers:
{ provide: APP_GUARD, useClass: ThrottlerGuard }
```

---

### Performance Tips

**Use Fastify** for raw throughput (2-3x more requests/sec than Express in benchmarks). Only blocked by Express-only middleware.

**Enable compression:**
```bash
npm install compression
```
```typescript
// main.ts (Express only)
import * as compression from 'compression';
app.use(compression());
```

**Paginate large queries.** Never return an unbounded list from a database:
```typescript
findAll(page = 1, limit = 20) {
  return this.repo.findAndCount({
    skip: (page - 1) * limit,
    take: limit,
  });
}
```

**Cache aggressively.** Wrap read-heavy endpoints with `CacheInterceptor`. Invalidate on write.

**Use `select` in TypeORM / Prisma.** Never `SELECT *` from tables with many columns or large text/blob fields.

---

### Development Workflow Tips

**Hot reload in dev:**
```bash
npm run start:dev   # uses ts-node-dev under the hood, watches for changes
```

**Debugging with VS Code** — add to `.vscode/launch.json`:
```json
{
  "type": "node",
  "request": "attach",
  "name": "Attach to NestJS",
  "port": 9229,
  "restart": true,
  "sourceMaps": true
}
```
Then run: `npm run start:debug`

**Use the Nest CLI's `--dry-run` flag** to preview what a generate command will create without writing files:
```bash
nest g resource products --dry-run
```

---

## 19. Study Path

Follow this order to go from zero to building real production apps.

### Phase 1: Foundations (Week 1)
1. Install Node.js 20+ and the Nest CLI. Run `nest new` and explore the generated files.
2. Read the entire `main.ts` and `app.module.ts` — understand the bootstrap flow.
3. **Build:** A simple CRUD REST API for a `Tasks` resource using in-memory arrays (no DB yet). Practice all HTTP methods, `@Param`, `@Query`, `@Body`.
4. Add `ValidationPipe` + DTOs with `class-validator`. Test with Postman or curl.

### Phase 2: Database (Week 2)
5. Set up PostgreSQL locally (Docker is easiest: `docker run -e POSTGRES_PASSWORD=pass -p 5432:5432 postgres`).
6. Add TypeORM to your Tasks app. Convert the in-memory array to a real `tasks` table.
7. Practice: entity relationships (a `User` has many `Tasks`), query builder, transactions.
8. **Alternative:** Redo step 6-7 using Prisma. Compare the DX. Pick one for future projects.

### Phase 3: Auth (Week 3)
9. Add a `Users` module with registration + hashed passwords.
10. Implement JWT authentication: login endpoint, JWT strategy, `JwtAuthGuard`.
11. Add the `@Public()` decorator for public routes. Protect all others.
12. Add role-based access control with `RolesGuard` and `@Roles()`.
13. **Build:** A full auth system. Test with Postman: register → login → use token → access protected route.

### Phase 4: The Middleware Stack (Week 4)
14. Write a request logging middleware. Apply it globally.
15. Write a `TransformInterceptor` that wraps all responses in `{ data: ..., timestamp: ... }`.
16. Write a global `HttpExceptionFilter` that formats errors consistently.
17. Write a `TimeoutInterceptor`.
18. Configure Swagger docs for your whole API.

### Phase 5: Advanced Features (Weeks 5-6)
19. Add `@nestjs/config` and move all secrets/config to `.env` with Joi validation.
20. Add caching to your most-read endpoints.
21. Add BullMQ queue for a background task (e.g., "send a welcome email after registration").
22. Add task scheduling (e.g., "clean up unverified accounts every night").
23. **Build:** A complete blog API (users, posts, comments, tags, auth, pagination, search, Swagger).

### Phase 6: Real-Time & Files (Week 7)
24. Add a WebSocket gateway for real-time notifications.
25. Add file upload for user avatars.
26. Serve uploaded files as static assets.

### Phase 7: Testing & Production (Week 8)
27. Write unit tests for every service using Jest mocks.
28. Write E2E tests for the critical flows (auth, CRUD).
29. Set up Docker: `Dockerfile` + `docker-compose.yml` for app + Postgres + Redis.
30. Configure for production: disable `synchronize`, add migrations, add Helmet + rate limiting.
31. **Final Project:** Deploy a production-ready NestJS API to Railway, Render, or a VPS.

---

### Build-to-Learn Projects (ordered by complexity)

| # | Project | Key Concepts Practiced |
|---|---------|----------------------|
| 1 | Task Manager API | Modules, Controllers, Services, DTOs, Validation, TypeORM/Prisma |
| 2 | Auth System | Passport, JWT, Guards, Decorators, Hashing |
| 3 | Blog Platform | Relations (user→posts→comments), pagination, search, Swagger |
| 4 | File Storage Service | File upload, static serving, stream responses |
| 5 | Real-Time Chat API | WebSocket Gateway, rooms, auth over WS |
| 6 | Job Board with Email Queue | BullMQ, nodemailer, task scheduling, background jobs |
| 7 | E-Commerce Backend | Complex relations, Stripe webhooks, inventory transactions, caching |
| 8 | Microservices System | Split the blog into Auth, Posts, and Notifications microservices using Redis transport |

---

### Quick reference: install map

```bash
# Core
npm install -g @nestjs/cli
nest new my-api

# Validation
npm install class-validator class-transformer @nestjs/mapped-types

# Config
npm install @nestjs/config joi

# TypeORM + PostgreSQL
npm install @nestjs/typeorm typeorm pg

# Prisma
npm install prisma @prisma/client && npx prisma init

# Mongoose
npm install @nestjs/mongoose mongoose

# Auth
npm install @nestjs/passport passport passport-jwt @nestjs/jwt bcrypt
npm install --save-dev @types/passport-jwt @types/bcrypt

# Swagger
npm install @nestjs/swagger

# WebSockets
npm install @nestjs/websockets @nestjs/platform-socket.io socket.io

# File uploads (Multer bundled — just needs types)
npm install --save-dev @types/multer

# Caching
npm install @nestjs/cache-manager cache-manager

# Scheduling
npm install @nestjs/schedule
npm install --save-dev @types/cron

# BullMQ queues
npm install @nestjs/bullmq bullmq ioredis

# Microservices
npm install @nestjs/microservices

# Security & performance
npm install helmet @nestjs/throttler compression
```

---

*NestJS documentation: https://docs.nestjs.com — bookmark it, it's excellent. All code in this guide is accurate as of NestJS 11 (2026). For any fast-moving APIs (BullMQ, Prisma, Passport), always cross-check the relevant package's own docs.*
