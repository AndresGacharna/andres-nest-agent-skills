---
name: andres-nestjs-architecture
description: Base architecture and conventions for NestJS 12 APIs (ESM + TypeORM + PostgreSQL). Use when creating or modifying modules, controllers, routes, services, DTOs, entities, enums, errors/exceptions, success messages, response format, paginated queries, file downloads, authorization (`@Auth`), Swagger docs, input sanitization (SQL, LIKE wildcards, free text), environment variables or database configuration in this project. Complements `nestjs-best-practices`; on a conflict, this skill wins.
---

# NestJS Architecture

Conventions so that every API reads the same way and scales without rewriting the base.

> Built with team conventions I worked with and the third-party skill `nestjs-best-practices` (`kadajett/agent-nestjs-skills`) as a reference. The decisions and the content of this skill are my own.

## Principles

1. **Thin controllers.** They take the request, validate it with DTOs and delegate. No `try/catch`, no response formatting and no manual logging.
2. **Services throw, they do not return errors.** Never `return new XException()` nor `{ success: false }`.
3. **Cross-cutting concerns live once, in `common/`.** Success messages, logging, errors and serialization are registered globally in `CommonModule`.
4. **Every module owns its own things.** Its errors, messages, constants, enums, DTOs and entities live inside its folder. `common/` never imports from a business module and never holds a module's texts or data.
5. **No `process.env` outside `src/config/`.** Everything is read through `ConfigService<EnvironmentVariables, true>`.

## Structure

```
src/
├── main.ts                      bootstrap: prefix, ValidationPipe, CORS, Swagger, port
├── app.module.ts                ConfigModule, TypeORM, CommonModule and business modules
├── config/
│   ├── env.validation.ts        EnvironmentVariables + validateEnv (fails at startup)
│   └── database.config.ts       buildTypeOrmOptions(config)
├── common/                      cross-cutting only, no business logic or texts
│   ├── common.module.ts         APP_FILTER + APP_INTERCEPTOR
│   ├── constants/               common.exception-response.ts, postgres-error-codes.ts
│   ├── decorators/              @ResponseMessage(), @SafeText(), @IsOptionalNotNull(),
│   │                            @ApiPagedBody(), @ApiMessageResponse(), @ApiPaginatedResponse(),
│   │                            @ApiErrorResponses()
│   ├── dto/                     PaginationDto, PagedDto, BaseFilterDto, OptionDto,
│   │                            EnumOptionDto, ApiErrorResponseDto
│   ├── exceptions/              DomainException
│   ├── filters/                 AllExceptionsFilter
│   ├── helpers/                 escapeLike(), sanitizeText(), containsMarkup(),
│   │                            toResponseDto(), toStreamableFile()
│   ├── interceptors/            LoggingInterceptor, ResponseMessageInterceptor
│   └── interfaces/              MessageResponse, ApiErrorResponse, PaginatedResult, GeneratedFile
└── <feature>/                   one module per domain (company, product, ...)
    ├── <feature>.module.ts
    ├── constants/<feature>.exception-response.ts
    ├── constants/<feature>.messages.ts
    ├── controllers/<feature>.controller.ts
    ├── dto/create-<feature>.dto.ts, update-<feature>.dto.ts, <feature>-filter.dto.ts
    ├── entities/<feature>.entity.ts
    ├── enums/                   optional: <name>.enum.ts with its <Name>EnumHelper
    ├── interfaces/              optional: internal types that never cross HTTP
    └── services/<feature>.service.ts
```

Full module template: [references/module-template.md](references/module-template.md).

## Rules

### ESM

- Relative imports end in `.js`: `import { X } from './x.service.js'`.
- TypeORM relations are typed with `Relation<T>`.
- Import order: external packages, blank line, relative imports.
- Types used only as types: `import type { X }`. They are erased at compile time and do not create dependencies between files.
- DON'T: Barrels (an `index.ts` that re-exports a folder): every import points at the file. A barrel drags in everything in the folder and hides cycles between modules, which in ESM end up as `Cannot access 'X' before initialization`.

### Controllers

- DO: `return this.service.method(...)`. The controller does not build the response.
- DO: **Queries** (`GET`, or `POST` with pagination or filters in the body): no `@ResponseMessage`. They return the result as is.
- DO: **Queries over `POST`:** add `@HttpCode(HttpStatus.OK)`. Without it, Nest answers `201 Created` on every `POST`. The paginated list is `POST /company/getAll`.
- DO: **Paginated queries document their body with `@ApiPagedBody(<Feature>PagedDto, { allFilters, ... })`**. `allFilters` is required and goes first: it is the example Swagger's "Try it out" loads, so it carries every filter of the DTO. Add one example per common search (`bySku`, `activeOnly`...). The filters are typed against the DTO, so a misspelled field does not compile.
- DO: **Actions** (create, update, delete): always `@ResponseMessage(<Feature>Messages.<ACTION>)`. They answer `{ message, data? }`.
- DO: `DELETE` answers `200` (Nest's default) so the message gets through. Never `@HttpCode(HttpStatus.NO_CONTENT)` on an action with a message: a `204` has no body.
- DO: `ParseUUIDPipe` on the `:id` params, DTOs with `class-validator` on `@Body()` and `@Query()`.
- DON'T: `try/catch` or `@Res()` (it skips interceptors and filters).
- DON'T: `@UseInterceptors(ClassSerializerInterceptor)` or `new Logger()` per controller: they are already global.
- DO: Routes use action naming: the controller in singular camelCase (`@Controller('costCenter')`) and every handler names its action in camelCase:
  - `GET getParams`: the lists (enums, select options) the forms need.
  - `GET getById/:id`: only when the detail shows more than the list. If a `getAll` row already carries everything the detail or edit form needs, skip it: the frontend reuses the row. Add it when the detail needs data the list leaves out (relations, long text, computed or history fields).
  - `POST getAll`: paginated and filtered list (`{ pagination, filter }` in the body).
  - `POST create`.
  - `PUT update/:id`: updates use `PUT` with a partial DTO, never `PATCH`.
  - `DELETE delete/:id`.
  - Other actions: `<verb>/:id` (`PUT deactivate/:id`, `POST import`).
- DON'T: REST-style routes with a bare `:id` or a plural resource (`GET /companies/:id`, `POST /companies`, `PATCH /companies/:id`, `POST /companies/search`).

**Downloads** (an Excel, a PDF)

- DO: `GET <verb>/:id`, no `@ResponseMessage`, and the handler wraps the service's file with `toStreamableFile()` from `common/helpers/to-streamable-file.helper.ts`. It is the one place where the controller turns the result into something HTTP-specific: it sets the `Content-Type` and the `Content-Disposition` (with the file name), which are transport details the service must not know. Document it with `@ApiProduces(<mime type>)` and `@ApiOkResponse({ schema: { type: 'string', format: 'binary' } })`.
  ```ts
  // A download: the .xlsx itself, not JSON, so no @ResponseMessage
  @Get('exportReport/:id')
  @ApiProduces(ReportTemplate.MIME_TYPE)
  @ApiOkResponse({ schema: { type: 'string', format: 'binary' } })
  async exportReport(@Param('id', ParseUUIDPipe) id: string) {
    return toStreamableFile(await this.reportService.exportReport(id));
  }
  ```
- DON'T: `@ResponseMessage` on a download: the interceptor would wrap the file in `{ message, data }` and the client would get JSON instead of the file.
- DO: Keep `exposedHeaders: ['Content-Disposition']` in the CORS setup of `main.ts`. The front runs on another origin and downloads with `fetch`/axios (to send the token); without it the browser hides the header and the front loses the file name. The front reads `filename*` (the exact UTF-8 name) and falls back to `filename`.

### Authorization (`@Auth`)

When the API has authentication, `@Auth()` (in `auth/decorators/auth.decorator.ts`) is a meta-decorator that sets the allowed roles, applies the guard and adds `ApiBearerAuth` plus the 401/403 Swagger responses.

**The usual case: one kind of account**

Most APIs have a single kind of user with a role enum. `@Auth` takes the roles as plain arguments, no array and no object:

```ts
@Get('getParams')
@Auth()                                      // any logged-in user, whatever the role
getParams() { ... }

@Post('create')
@Auth(RoleEnum.Admin, RoleEnum.Leader)       // only these roles
@ResponseMessage(CompanyMessages.CREATED)
create(@Body() dto: CreateCompanyDto) { ... }
```

- DO: Every handler carries its own `@Auth`, placed after the route decorator (`@Get`/`@Post`/`@Put`) and its `@HttpCode` if any, before `@ResponseMessage`, `@UseInterceptors` and the Swagger decorators.
- DON'T: `@Auth` on the `@Controller` class. A handler-level `@Auth` **replaces** the class one (`getAllAndOverride`), it does not add to it; with one per handler, reading the handler is enough to know who gets in.
- DO: `@Auth()` with no roles when every logged-in user may do it; list the roles when only some may. The admin role is always listed explicitly: no implicit bypass in the guard.
- DO: The guard rejects the enum's placeholder value (`Unknown = 0` or similar) with a 403, even on `@Auth()`: an account without a real role reaches nothing.
- DO: Only the endpoints that must work without a token (login, SSO callback, register, forgot/reset password) go without `@Auth`. Every other endpoint has one: a handler without `@Auth` is public.
- DON'T: Roles in the enum that no endpoint uses. An unused role only adds doubt about who can do what; add it when the first endpoint needs it.

**Variant: two kinds of accounts**

Some APIs serve two audiences with separate logins, for example internal users with roles plus external partner accounts (suppliers, clients) with no roles. The token carries the audience (`aud`) and `@Auth` takes one key per audience:

```ts
@Auth({ user: [] })                          // any internal user
@Auth({ user: [UserRoleEnum.Admin] })        // only these internal roles
@Auth({ partner: true })                     // only partner accounts
@Auth({ user: [], partner: true })           // both
```

- A missing key means that audience cannot enter; `user: []` means any internal role (same meaning as `@Auth()` above).
- DO: A partner account's own id (`partnerId`) comes from the token (`@GetUser()`), never from the route. When both audiences reach the same endpoint, the service scopes the data to that id when it is set.
- Everything else above applies unchanged: one `@Auth` per handler, explicit admin, placeholder role rejected, only the login flows without it.

### Services

- DO: Return the created or updated resource (or `void` when there is nothing to return). The success message is added by the controller with `@ResponseMessage`, never by the service.
- DO: Missing resource: `throw new DomainException(HttpStatus.NOT_FOUND, <Feature>ExceptionResponse.NOT_FOUND)`.
- DO: Broken business rule: `throw new DomainException(HttpStatus.<STATUS>, <Feature>ExceptionResponse.<CASE>)`.
- DO: A `try/catch` only when something is going to be **recovered** (a retry, a fallback, processing a partial batch). If it is only going to rethrow, do not write it.
- DO: `return` for expected results even when they are not the happy path (`null`, `boolean`, partial results); `throw` only when the request cannot continue.
- DO: A service that builds a file returns a `GeneratedFile` (`common/interfaces/generated-file.interface.ts`): `{ fileName, mimeType, content: Buffer }`. It knows nothing about HTTP, so the same file can go to a download (the controller) or to an email attachment (another service). The file layout (sheet, columns, MIME type, extension) lives in a `constants/<name>.template.ts`, and the writing in its own `<name>-writer.service.ts`, as the reader of an upload does.
- DON'T: `new StreamableFile(...)`, `Content-Disposition` or `res.setHeader()` in a service: they are HTTP, and they tie the file to one way of sending it.
- DON'T: `return '<Resource> created successfully'` (or a `ServiceResponse` class): the message belongs in the controller.
- DON'T: `throw new BadRequestException(error.message)` with the message of an unknown error: it leaks internal details.
- DON'T: Comparing `error.message === '...'` to decide: if the flow depends on that case, the service returns it as a result (`null`, `boolean`) instead of throwing it.

### Errors

- `DomainException(status, message)`: the status from `HttpStatus` and the text the frontend shows.
- Per-module message catalog in `<feature>/constants/<feature>.exception-response.ts`. **Every** user-facing text goes there, dynamic ones included: a text with values is a function in the catalog.
  ```ts
  export const ProductExceptionResponse = {
    NOT_FOUND: 'Product not found',
    SKU_TAKEN: (sku: string) => `A product with SKU ${sku} already exists`,
    OVER_STOCK: (sku: string, available: number) =>
      `Product ${sku} only has ${available} units available`,
  } as const;
  ```
  ```ts
  throw new DomainException(
    HttpStatus.CONFLICT,
    ProductExceptionResponse.SKU_TAKEN(dto.sku),
  );
  ```
- DON'T: Inline texts in a service or controller, not even single-use or dynamic ones (`` new DomainException(HttpStatus.NOT_FOUND, `Product ${id} not found`) ``). Internal log lines (`this.logger.*`) are not user-facing and stay inline.
- When the error wraps another one (an HTTP call, a library), pass the original as the cause: `new DomainException(status, message, { cause: error })`. It is not shown to the client, but it helps to decide and to debug.
- The error response carries no `code`. To branch on an error, the frontend uses `statusCode`, never `message`.
- `common/constants/common.exception-response.ts` is only for cross-cutting messages. Do not add business messages there.
- Postgres duplicates (23505) and foreign keys (23503) are already translated to a 409 with a generic message in the filter. Validate beforehand in the service only when a module-specific message is needed.
- 5xx exceptions that are not a `DomainException` answer with a generic message: the detail stays in the log.

### Success messages

- Per-module catalog in `<feature>/constants/<feature>.messages.ts`:
  ```ts
  export const SupplierMessages = {
    CREATED: 'Supplier created successfully',
    UPDATED: 'Supplier updated successfully',
    DELETED: 'Supplier deleted successfully',
  } as const;
  ```
- Every action (create, update, delete, processes that change data) carries a message. Queries do not.
- When the text **depends on the result** (for example "3 created, 2 skipped"), it is not a fixed message: the summary goes in what the service returns (`{ created, skipped }`) and ends up in `data`.
- The catalogs are written in the language the frontend shows; pick one per API and keep it. Validation messages from `class-validator` stay in English.

#### Example in a controller

```ts
@Controller('supplier')
export class SupplierController {
  constructor(private readonly supplierService: SupplierService) {}

  // Query over GET: no decorator → answers the result as is
  @Get('getParams')
  getParams() {
    return this.supplierService.getParams();
  }

  // Query over POST: no decorator and with @HttpCode(200) → { page, size, count, rows }
  @Post('getAll')
  @HttpCode(HttpStatus.OK)
  getAll(@Body() pagedDto: SupplierPagedDto) {
    return this.supplierService.getAll(pagedDto);
  }

  // Action: → { message, data }
  @Post('create')
  @ResponseMessage(SupplierMessages.CREATED)
  create(@Body() dto: CreateSupplierDto) {
    return this.supplierService.create(dto);
  }

  // Non-CRUD action: same pattern
  @Put('deactivate/:id')
  @ResponseMessage(SupplierMessages.DEACTIVATED)
  deactivate(@Param('id', ParseUUIDPipe) id: string) {
    return this.supplierService.deactivate(id);
  }

  // Action with no return value: → { message }
  @Delete('delete/:id')
  @ResponseMessage(SupplierMessages.DELETED)
  remove(@Param('id', ParseUUIDPipe) id: string) {
    return this.supplierService.remove(id);
  }

  // Action with a summary: the message is fixed and the detail goes in data
  // → { message, data: { created: 3, skipped: 2 } }
  @Post('import')
  @ResponseMessage(SupplierMessages.IMPORTED)
  import(@Body() dto: ImportSuppliersDto) {
    return this.supplierService.import(dto);
  }
}
```

- The decorator goes on the **handler**, not on the `@Controller`: a controller mixes queries and actions, and at class level the message would also apply to the `GET`s.
- The `import` service returns `{ created, skipped }` and builds no text at all.

### Response format (never built by hand)

| Endpoint type                        | Response                                                          |
| ------------------------------------ | ----------------------------------------------------------------- |
| Query (`GET` or search over `POST`)  | The result as is: entity, array or `{ page, size, count, rows }`  |
| Action that returns something        | `{ "message": "...", "data": ... }`                               |
| Action that returns nothing (`void`) | `{ "message": "..." }`                                            |
| Download (file)                      | The file bytes, with `Content-Type` and `Content-Disposition`     |
| Error                                | `{ "statusCode", "message", "timestamp", "path" }`                |

- A download's errors are still JSON: the service throws (404, 400) before the file exists, so the global filter answers as for any other endpoint. That holds because the file is built whole in memory first; a stream that fails halfway can no longer change its status. `ClassSerializerInterceptor` and `LoggingInterceptor` let a `StreamableFile` through untouched.
- An error `message` is a `string[]` when the error comes from `ValidationPipe`. `path` does not include the query string.
- Do not add `timestamp` or `path` to successful responses: the client already knows them. On errors they help to match a reported error with the logs.
- Sensitive fields on entities: `@Exclude()` from `class-transformer`. The global `ClassSerializerInterceptor` applies it on queries and actions.
- **Entity or response DTO:**
  - Simple CRUD: return the entity, with `@Exclude()` on whatever must not go out.
  - Response DTO (`<feature>/dto/<feature>-response.dto.ts`) when the resource has computed or relation fields, or sensitive data (users, credentials). There, an explicit list of what goes out is safer than remembering the `@Exclude()` on every new column.
  - DO: Build it with `toResponseDto(Dto, entity)` (or a list of entities) from `common/helpers/to-response-dto.helper.ts`. It wraps `plainToInstance` with `excludeExtraneousValues: true`, so **only the `@Expose()` fields go out**: a column the DTO does not declare (an internal `number`, a `password`) never leaks.
  - DO: `@Expose()` on every field of a response DTO, nested DTOs included (`@Expose()` + `@Type(() => ItemDto)`). A field without it is silently left out.
  - DO: A field that comes from a relation or is computed carries `@Transform(({ obj }: { obj: Entity }) => ..., { toClassOnly: true })`. Typing `obj` with the entity brings back the compiler's checks inside the function.
  - DON'T: `@Transform` without `{ toClassOnly: true }`. The global `ClassSerializerInterceptor` runs `instanceToPlain` on the DTO afterwards, and a two-way `@Transform` runs again over the DTO, where the relation (`obj.company`) no longer exists: a 500.
  - DO: A computed value used beyond one response (an email, an export) is a getter on the entity (`get code()`), and the DTO only takes it with `@Expose()`, without `@Transform`: the formula lives once.
  - DON'T: Pass a spread (`{ ...entity, extra }`) to `toResponseDto`: a spread drops the entity's getters. When two endpoints compute a field differently (a count from `loadRelationCountAndMap` in the list, from the loaded items in the detail), the detail DTO redeclares the field with its own `@Transform`.
  - DON'T: Hand-built mapping functions (`toSupplierResponse(entity)`) or an automapper.
  - DON'T: `class-validator` decorators on a response DTO: validation only runs over what comes in.

### Swagger

- Docs are served at `/api/docs` outside production only: they expose every route and payload.
- DO: `@ApiTags` on the controller and `@ApiOperation({ summary, description })` on each handler; the description states what the filters match and the uniqueness rules (409s).
- DO: Document the envelopes with the `common/decorators/api-responses.decorator.ts` helpers, because Swagger cannot read generic classes: `@ApiMessageResponse(status, Model?)` for actions, `@ApiPaginatedResponse(Model)` for `getAll`, `@ApiErrorResponses(...statuses)` for the error statuses the handler can answer.
- DO: Every field of an entity that goes out, and of every DTO, has its `@ApiProperty` / `@ApiPropertyOptional`. A field with `@Exclude()` has none.
- DO: Neutral, made-up example data that fits the field: `Example Company Inc.`, `ana.paz@example.com`, `Football, size 5, white`. DON'T: real companies, brands, stores, products or people.
- DO: A list of the shared `OptionDto` or `EnumOptionDto` sets its own `example` on the property: the class example is generic and would show the same text in every list. For an enum, use the helper's own list (`example: ProductCategoryEnumHelper.GetList()`), so the docs never drift from the API; for entities, a value that matches what the service builds.

### Enums

- An enum lives in `<feature>/enums/<name>.enum.ts` next to its `<Name>EnumHelper`, a class with the texts in a `Record<Enum, string>` and static methods: `GetText(value)` and `GetList()`, which returns the `EnumOptionDto[]` (`{ id, value }`) that `getParams` sends to the forms.
- DO: Numeric enums. The text the user sees comes from the helper, never from the enum key.
- DO: When the enum has a placeholder (`Unknown = 0`), `GetList()` leaves it out: it is never an option.

### DTOs, interfaces and where each type lives

- **A class in `dto/`** when something uses it at runtime:
  - what comes in and is validated (`@Body()`, `@Query()`, `@Param()`) with `class-validator`;
  - what is transformed with `class-transformer` (`@Expose({ name: 'access_token' })` when reading the response of an external service, `@Exclude()`);
  - what Swagger will document (`@ApiProperty` needs classes).
- **`null` in write DTOs.** `@IsOptional()` skips both `undefined` and `null`, so a `null` passes validation and fails on a NOT NULL column as a 500.
  - DO: `UpdateDto extends PartialType(CreateDto, { skipNullProperties: false })` from `@nestjs/swagger` (it also copies the Swagger metadata): only an omitted field is skipped, a `null` gets a 400.
  - DO: An optional field of a create or update DTO that goes to a NOT NULL column (`isActive`) uses `@IsOptionalNotNull()` from `common/decorators/is-optional-not-null.decorator.ts`, not `@IsOptional()`.
  - DO: Keep `@IsOptional()` where `null` is a valid value (`description: null` clears a nullable column, `approverId: null` removes the approver) and on filter DTOs, where `null` means "not sent".
- **Paginated queries:** the body is `<Feature>PagedDto extends PagedDto<<Feature>FilterDto>`, both in `dto/<feature>-filter.dto.ts`. It redeclares `filter` with its own `@Type(() => <Feature>FilterDto)`, because class-transformer cannot resolve a generic at runtime. The service returns a `PaginatedResult<T>` (`{ page, size, count, rows }`); `size` is capped by `MAX_PAGE_SIZE`.
- **An interface in `interfaces/`** when it is only a compile-time type: internal shapes that are never an incoming or outgoing body (data built between two methods, the envelope produced by an interceptor or a filter). A class with no decorators adds nothing.
- The `dto/` folder is singular. Names: `create-<feature>.dto.ts`, `<feature>-response.dto.ts`, `<feature>-params-response.dto.ts`, and for external API responses the provider's name (`<provider>-token-response.dto.ts`).
- **A type lives with whoever defines it, not with whoever uses it first.** If something in `common/` produces a shape (the `{ token, nonce, exp }` envelope of an encryption utility, the `MessageResponse` from the interceptor), its interface goes in `common/` next to that code, even if today only `auth` consumes it. That way `common/` never imports from a business module and the `common → auth → common` cycle never forms.
- When two business modules need a type, the module that owns the concept exports it and the other one imports it in that direction. It moves to `common/` only if it really is cross-cutting.
- What belongs to another integration does not go into `auth` just because it looks similar: the external client login (`clientId`/`clientSecret`) gets its own module.
- `MessageResponse` and `ApiErrorResponse` are interfaces because nobody instantiates or validates them; Swagger reads `ApiErrorResponseDto`, which implements the interface.

### Input sanitization

Three different risks, each solved in a different place. Do not mix them.

| Risk | Where it is solved | Tool |
| --- | --- | --- |
| SQL injection | The query | Named parameters (`:name`). Already covered by TypeORM |
| `LIKE` wildcards (`%`, `_`) | The query | `escapeLike()` from `common/helpers/escape-like.helper.ts` |
| HTML and invisible characters in free text | The incoming DTO | `@SafeText(maxLength)` from `common/decorators/safe-text.decorator.ts` |

**SQL**

- DO: Every value goes to the query as a named parameter: `.andWhere('product.sku = :sku', { sku })`.
- DON'T: Build SQL with template strings or concatenation of anything that comes from the request (`` `product.sku = '${sku}'` ``), not even in `orderBy`: an order column picked by the client is mapped through a fixed whitelist.

**`LIKE` / `ILIKE` wildcards**

- DO: Every `ILIKE` that receives user text escapes it first: `{ textFilter: `%${escapeLike(textFilter)}%` }`. `escapeLike` puts a backslash before `\`, `%` and `_` (Postgres' default `LIKE` escape character).
- DO: Escape in the query, not in the DTO: searching for "50%" is legitimate, the `%` only must not act as a wildcard.
- DON'T: Pass the raw text between `%...%`. `%%%%%` (or `_____`) passes any minimum length and returns the whole table.

**Free text (`@SafeText`)**

- DO: Free-text strings that come in (names, business names, addresses, observations, SKU, `textFilter`) use `@SafeText(maxLength)`. It is a meta-decorator (`applyDecorators`) that:
  - normalizes to NFC, then removes control characters (C0 and C1, keeping `\t`, `\n`, `\r`), zero-width characters (`​`, `⁠`, `﻿`) and bidi controls (`‪`-`‮`, `⁦`-`⁩`), and trims, with a `@Transform` that only touches strings. Bidi controls reorder how text is displayed, so a stored value can read differently on screen, in an email or in an Excel export (`"Invoice‮fdp.exe"` shows as `"Invoiceexe.pdf"`). ZWJ and ZWNJ (`‍`, `‌`) are kept: emoji sequences and some scripts need them;
  - validates `@IsString()` and `@MaxLength(maxLength)`;
  - **rejects** `<` and `>` with `@Matches`: a 400 from the `ValidationPipe`, with the message in English like every validation message.
- DO: **Filters too.** `BaseFilterDto.textFilter` already carries `@SafeText(100)`, so a `<Feature>FilterDto extends BaseFilterDto` inherits it: do not redeclare `textFilter`. Every other text field of a filter DTO (`sku`, `name`, `code`) gets `@IsOptional()` + `@SafeText(n)`. A filter field that is a partial email keeps its own `@Transform` + `@IsString()` + `@MaxLength()` (not `@SafeText`, not `@IsEmail`: it is a fragment), and like every filter its value still goes through `escapeLike()` in the query. A filter needs both: `@SafeText` cleans what comes in, `escapeLike` neutralizes the wildcards in the query.
- DO: `@SafeText` replaces the `@Transform(trim)` + `@IsString()` + `@MaxLength()` trio; do not repeat them. `@IsOptional()`, `@IsNotEmpty()`, `@MinLength()` and the Swagger `@ApiProperty` stay on the field next to it.
- DO: A search with a minimum length (to avoid "a" returning everything) puts the number in a `<Feature>Limits` constant of the module (`constants/<feature>.limits.ts`) and uses it in `@MinLength()` and in the Swagger description. `@SafeText` trims before validation, so `"a    "` does not pass.
- DON'T: Clean HTML silently (`sanitize-html` with `allowedTags: []`, stripping tags). The `security-sanitize-output` rule of `nestjs-best-practices` proposes it; here this skill wins: plain-text fields must not contain markup, and a 400 tells the user what was not saved, instead of storing `"Dumbbell "` from `"Dumbbell <5kg>"`. No dependency is added for this.
- DON'T: `@SafeText` on emails, passwords, UUIDs, enums, numbers or dates: they already have their own validator (`@IsEmail`, `ParseUUIDPipe`, `@IsEnum`...), and a password may legitimately contain `<` or `>`.
- The frontend still escapes on render: `@SafeText` protects what comes in from now on, not what is already in the database.

**Text that does not come through a DTO (Excel imports, files)**

- DO: The cleaning lives in `common/helpers/sanitize-text.helper.ts`: `sanitizeText(value)` (the same NFC + control + zero-width + bidi + trim that `@SafeText` applies) and `containsMarkup(value)` (`<` or `>`). `@SafeText` uses them, so there is one implementation.
- DO: A file reader cleans every cell with `sanitizeText()` before validating it, and a text column that is stored rejects markup as a row error, in the catalogs' language and next to the other row errors (the text is a function in the module's catalog: `ImportExceptionResponse.MARKUP_NOT_ALLOWED(COLUMNS.name.header)`). A column only used to look something up (a SKU to match) is cleaned, not rejected.
- DON'T: Store a cell's raw `cell.text`: it bypasses `@SafeText` entirely.

### Configuration

- A new variable: add it to `EnvironmentVariables`, with an **explicit** type even when it has a default value (`PORT: number = 3033`), and to `.env.template`.
- Read it with `config.get('VAR', { infer: true })` using `ConfigService<EnvironmentVariables, true>`.
- Configuration of an external module: `forRootAsync({ inject: [ConfigService], useFactory })` with the factory in `src/config/`.

### Database

- `pg` pool options: `poolSize` (mapped to `max`) and `extra` with node-postgres options. Never MySQL options (`connectionLimit`, `timezone`).
- A single table: the injected TypeORM repository. Several writes that must go together: `dataSource.transaction(...)`.
- Avoid N+1: `relations` or a `QueryBuilder` with `leftJoinAndSelect`.

## Checklist for a new module

1. `nest g resource <feature>` (or copy the template) and move the files into `controllers/`, `services/`, `dto/`, `entities/`.
2. Create `constants/<feature>.exception-response.ts` with at least the `NOT_FOUND` message (dynamic texts as functions), and `constants/<feature>.messages.ts` with one message per action.
3. DTOs with `class-validator`; `UpdateDto extends PartialType(CreateDto, { skipNullProperties: false })` from `@nestjs/swagger`, and `@IsOptionalNotNull()` on optional fields that go to a NOT NULL column (see DTOs). Free-text fields with `@SafeText(maxLength)` and every `ILIKE` with `escapeLike()` (see Input sanitization). `<Feature>PagedDto` + `<Feature>FilterDto extends BaseFilterDto` for `getAll`.
4. A service that throws `DomainException(HttpStatus.X, <Feature>ExceptionResponse.Y)` and returns entities (or response DTOs built with `toResponseDto`).
5. A controller with the action routes (`getParams`, `getAll`, `create`, `update/:id`; `getById/:id` only when the list lacks the detail's data), no try/catch and no manual formatting: `@ResponseMessage` on every action, `@HttpCode(HttpStatus.OK)` on every query over `POST`, `@Auth` on every handler when the API has authentication, and the Swagger decorators.
6. `TypeOrmModule.forFeature([Entity])` in the module; export the service only when another module uses it.
7. Register the module in `app.module.ts`.
8. `pnpm exec tsc --noEmit`, `pnpm run lint` and `pnpm run build` with no errors.

## Related `nestjs-best-practices` rules

`arch-feature-modules`, `arch-avoid-circular-deps`, `arch-module-sharing`, `error-use-exception-filters`, `error-throw-http-exceptions`, `api-use-interceptors`, `api-use-dto-serialization`, `api-use-pipes`, `security-validate-all-input`, `security-sanitize-output` (adapted: reject instead of strip, see Input sanitization), `devops-use-config-module`, `db-avoid-n-plus-one`, `db-use-transactions`.
