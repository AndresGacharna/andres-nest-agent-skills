---
name: andres-nestjs-architecture
description: Arquitectura base y convenciones para APIs NestJS 12 (ESM + TypeORM + PostgreSQL). Usar al crear o modificar módulos, controladores, servicios, DTOs, entidades, errores/excepciones, mensajes de éxito, formato de respuestas, consultas paginadas, variables de entorno o configuración de base de datos en este proyecto. Complementa a `nestjs-best-practices`; ante un conflicto, manda esta skill.
---

# NestJS Architecture

Convenciones propias para que todas las APIs se lean igual y escalen sin reescribir la base.

> Construidas tomando como referencia convenciones de equipo y la skill de terceros `nestjs-best-practices` (`kadajett/agent-nestjs-skills`). Las decisiones y el contenido de esta skill son propios.

## Principios

1. **Controladores delgados.** Reciben la petición, validan con DTOs y delegan. Sin `try/catch`, sin formatear respuestas y sin logs manuales.
2. **Los servicios lanzan, no retornan errores.** Nunca `return new XException()` ni `{ success: false }`.
3. **Lo transversal va una sola vez, en `common/`.** Mensajes de éxito, logging, errores y serialización se registran globalmente en `CommonModule`.
4. **Cada módulo es dueño de lo suyo.** Sus errores, mensajes, constantes, DTOs y entidades viven dentro de su carpeta. `common/` nunca importa de un módulo de negocio ni guarda textos o datos de un módulo.
5. **Nada de `process.env` fuera de `src/config/`.** Todo se lee con `ConfigService<EnvironmentVariables, true>`.

## Estructura

```
src/
├── main.ts                      bootstrap: prefijo, ValidationPipe, CORS, puerto
├── app.module.ts                ConfigModule, TypeORM, CommonModule y módulos de negocio
├── config/
│   ├── env.validation.ts        EnvironmentVariables + validateEnv (falla al arrancar)
│   └── database.config.ts       buildTypeOrmOptions(config)
├── common/                      solo lo transversal, sin lógica ni textos de negocio
│   ├── common.module.ts         APP_FILTER + APP_INTERCEPTOR
│   ├── constants/               common.exception-response.ts, postgres-error-codes.ts
│   ├── decorators/              @ResponseMessage()
│   ├── exceptions/              DomainException
│   ├── filters/                 AllExceptionsFilter
│   ├── interceptors/            LoggingInterceptor, ResponseMessageInterceptor
│   └── interfaces/              MessageResponse, ApiErrorResponse
└── <feature>/                   un módulo por dominio (company, supplier, ...)
    ├── <feature>.module.ts
    ├── constants/<feature>.exception-response.ts
    ├── constants/<feature>.messages.ts
    ├── controllers/<feature>.controller.ts
    ├── dto/create-<feature>.dto.ts, update-<feature>.dto.ts
    ├── entities/<feature>.entity.ts
    ├── interfaces/              opcional: tipos internos que no cruzan HTTP
    └── services/<feature>.service.ts
```

Plantilla completa de un módulo: [references/module-template.md](references/module-template.md).

## Reglas

### ESM

- Los imports relativos terminan en `.js`: `import { X } from './x.service.js'`.
- Las relaciones de TypeORM se tipan con `Relation<T>`.
- Orden de imports: paquetes externos, línea en blanco, imports relativos.
- Tipos que solo se usan como tipo: `import type { X }`. Se borran al compilar y no crean dependencias entre archivos.
- ❌ Barrels (`index.ts` que reexporta una carpeta): cada import apunta al archivo. Un barrel arrastra todo lo de la carpeta y esconde ciclos entre módulos, que en ESM terminan en `Cannot access 'X' before initialization`.

### Controladores

- ✅ `return this.service.metodo(...)`. El controlador no arma la respuesta.
- ✅ **Consultas** (`GET`, o `POST` con paginación o filtros en el body): sin `@ResponseMessage`. Devuelven el resultado tal cual.
- ✅ **Consultas por `POST`:** agregar `@HttpCode(HttpStatus.OK)`. Sin él, Nest responde `201 Created` en todo `POST`. Nombre de ruta descriptivo: `POST /companies/search`.
- ✅ **Acciones** (crear, actualizar, eliminar): siempre `@ResponseMessage(<Feature>Messages.<ACCION>)`. Responden `{ message, data? }`.
- ✅ `DELETE` responde `200` (default de Nest) para que el mensaje llegue. Nunca `@HttpCode(HttpStatus.NO_CONTENT)` en una acción con mensaje: un `204` no tiene body.
- ✅ `ParseUUIDPipe` en los `:id`, DTOs con `class-validator` en `@Body()` y `@Query()`.
- ❌ `extends ControllerBase`, `manageResponse`, `try/catch` o `@Res()` (se salta interceptores y filtros).
- ❌ `@UseInterceptors(ClassSerializerInterceptor)` o `new Logger()` por controlador: ya son globales.
- ❌ Rutas que repiten el verbo HTTP en acciones (`POST /create`, `PUT /update/:id`): usar `POST /companies`, `PATCH /companies/:id`.

### Servicios

- ✅ Devolver el recurso creado o actualizado (o `void` si no hay nada que devolver). El mensaje de éxito lo agrega el controlador con `@ResponseMessage`, nunca el servicio.
- ✅ Recurso inexistente: `throw new DomainException(HttpStatus.NOT_FOUND, <Feature>ExceptionResponse.NOT_FOUND)`.
- ✅ Regla de negocio violada: `throw new DomainException(HttpStatus.<STATUS>, <Feature>ExceptionResponse.<CASO>)`.
- ✅ Un `try/catch` solo si se va a **recuperar** algo (reintento, fallback, procesar un lote parcial). Si solo se va a relanzar, no se escribe.
- ✅ `return` para resultados esperados aunque no sean el caso feliz (`null`, `boolean`, resultados parciales); `throw` solo cuando la petición no puede continuar.
- ❌ `return '<Recurso> creado exitosamente'` (o una clase `ServiceResponse`): el mensaje va en el controlador.
- ❌ `throw new BadRequestException(error.message)` con el mensaje de un error desconocido: filtra detalles internos.
- ❌ Comparar `error.message === '...'` para decidir: si el flujo depende de ese caso, el servicio lo devuelve como resultado (`null`, `boolean`) en vez de lanzarlo.

### Errores

- `DomainException(status, message)`: el status de `HttpStatus` y el texto que el frontend muestra.
- Catálogo de mensajes por módulo en `<feature>/constants/<feature>.exception-response.ts`:
  ```ts
  export const SupplierExceptionResponse = {
    NOT_FOUND: 'Supplier not found',
    TAX_ID_TAKEN: 'A supplier with this tax id already exists',
  } as const;
  ```
  ```ts
  throw new DomainException(HttpStatus.NOT_FOUND, SupplierExceptionResponse.NOT_FOUND);
  ```
- El mensaje va en el catálogo cuando se repite o es parte del contrato con el frontend. Si es de un solo uso o dinámico, se escribe en línea: `new DomainException(HttpStatus.NOT_FOUND, \`Supplier ${id} not found\`)`.
- Si el error envuelve otro (una llamada HTTP, una librería), pasar el original como causa: `new DomainException(status, message, { cause: error })`. No se muestra al cliente, pero sirve para decidir y para depurar.
- La respuesta de error no lleva `code`. Para decidir según el error, el frontend usa `statusCode`, nunca `message`.
- `common/constants/common.exception-response.ts` es solo para mensajes transversales. No agregar mensajes de negocio ahí.
- Duplicados (23505) y claves foráneas (23503) de Postgres ya se traducen a 409 con un mensaje genérico en el filtro. Validar antes en el servicio solo si se necesita un mensaje específico del módulo.
- Las excepciones 5xx que no son `DomainException` responden un mensaje genérico: el detalle queda en el log.

### Mensajes de éxito

- Catálogo por módulo en `<feature>/constants/<feature>.messages.ts`:
  ```ts
  export const SupplierMessages = {
    CREATED: 'Supplier created successfully',
    UPDATED: 'Supplier updated successfully',
    DELETED: 'Supplier deleted successfully',
  } as const;
  ```
- Toda acción (crear, actualizar, eliminar, procesos que modifican datos) lleva mensaje. Las consultas no.
- Si el texto **depende del resultado** (por ejemplo "3 creados, 2 omitidos"), no es un mensaje fijo: el resumen va en lo que devuelve el servicio (`{ created, skipped }`) y queda en `data`.

#### Ejemplo en un controlador

```ts
@Controller('suppliers')
export class SupplierController {
  constructor(private readonly supplierService: SupplierService) {}

  // Consulta por GET: sin decorador → responde la entidad tal cual
  @Get(':id')
  findOne(@Param('id', ParseUUIDPipe) id: string) {
    return this.supplierService.findOne(id);
  }

  // Consulta por POST: sin decorador y con @HttpCode(200) → { page, size, count, rows }
  @Post('search')
  @HttpCode(HttpStatus.OK)
  search(@Body() pagedDto: SupplierPagedDto) {
    return this.supplierService.search(pagedDto);
  }

  // Acción: → { message, data }
  @Post()
  @ResponseMessage(SupplierMessages.CREATED)
  create(@Body() dto: CreateSupplierDto) {
    return this.supplierService.create(dto);
  }

  // Acción que no es CRUD: mismo patrón
  @Patch(':id/deactivate')
  @ResponseMessage(SupplierMessages.DEACTIVATED)
  deactivate(@Param('id', ParseUUIDPipe) id: string) {
    return this.supplierService.deactivate(id);
  }

  // Acción sin retorno: → { message }
  @Delete(':id')
  @ResponseMessage(SupplierMessages.DELETED)
  remove(@Param('id', ParseUUIDPipe) id: string) {
    return this.supplierService.remove(id);
  }

  // Acción con resumen: el mensaje es fijo y el detalle va en data
  // → { message, data: { created: 3, skipped: 2 } }
  @Post('import')
  @ResponseMessage(SupplierMessages.IMPORTED)
  import(@Body() dto: ImportSuppliersDto) {
    return this.supplierService.import(dto);
  }
}
```

- El decorador se pone en el **handler**, no en el `@Controller`: un controlador mezcla consultas y acciones, y a nivel de clase el mensaje se aplicaría también a los `GET`.
- El servicio de `import` devuelve `{ created, skipped }` y no arma ningún texto.

### Formato de respuestas (no se construye a mano)

| Tipo de endpoint | Respuesta |
|---|---|
| Consulta (`GET` o `POST` de búsqueda) | El resultado tal cual: entidad, arreglo o `{ page, size, count, rows }` |
| Acción que devuelve algo | `{ "message": "...", "data": ... }` |
| Acción que no devuelve nada (`void`) | `{ "message": "..." }` |
| Error | `{ "statusCode", "message", "timestamp", "path" }` |

- `message` de error es un `string[]` cuando el error viene de `ValidationPipe`. `path` no incluye el query string.
- No agregar `timestamp` ni `path` a las respuestas exitosas: el cliente ya los conoce. En los errores sirven para cruzar un error reportado con los logs.
- Campos sensibles en entidades: `@Exclude()` de `class-transformer`. El `ClassSerializerInterceptor` global lo aplica en consultas y acciones.
- **Entidad o response DTO:**
  - CRUD simple: devolver la entidad, con `@Exclude()` en lo que no debe salir.
  - Response DTO (`<feature>/dto/<feature>-response.dto.ts`) cuando el recurso tiene campos calculados o de relaciones, o datos sensibles (usuarios, credenciales). Ahí una lista explícita de lo que sale es más segura que acordarse del `@Exclude()` en cada columna nueva.
  - Mapeo con una función (`toSupplierResponse(entity)`) o `plainToInstance(Dto, entity, { excludeExtraneousValues: true })` + `@Expose()`. Sin automapper.
  - ❌ Decoradores de `class-validator` en un DTO de respuesta: la validación solo corre sobre lo que entra.

### DTOs, interfaces y dónde vive cada tipo

- **Clase en `dto/`** cuando algo la usa en tiempo de ejecución:
  - lo que entra y se valida (`@Body()`, `@Query()`, `@Param()`) con `class-validator`;
  - lo que se transforma con `class-transformer` (`@Expose({ name: 'access_token' })` al leer la respuesta de un servicio externo, `@Exclude()`);
  - lo que documentará Swagger (`@ApiProperty` necesita clases).
- **Interfaz en `interfaces/`** cuando es solo un tipo en tiempo de compilación: formas internas que nunca son body de entrada ni de salida (datos armados entre dos métodos, el sobre que produce un interceptor o un filtro). Una clase sin decoradores no aporta nada.
- Carpeta `dto/` en singular. Nombres: `create-<feature>.dto.ts`, `<feature>-response.dto.ts`, y para respuestas de APIs externas el nombre del proveedor (`<provider>-token-response.dto.ts`).
- **El tipo vive con quien lo define, no con quien lo usa primero.** Si algo de `common/` produce una forma (el sobre `{ token, nonce, exp }` de una utilidad de cifrado, el `MessageResponse` del interceptor), su interfaz va en `common/` junto a ese código, aunque hoy solo la consuma `auth`. Así `common/` nunca importa de un módulo de negocio y no se forma el ciclo `common → auth → common`.
- Si un tipo lo necesitan dos módulos de negocio, lo exporta el módulo dueño del concepto y el otro lo importa en esa dirección. Se mueve a `common/` solo si de verdad es transversal.
- Lo de otra integración no se mete en `auth` por parecerse: el login de clientes externos (`clientId`/`clientSecret`) va en su propio módulo.
- `MessageResponse` y `ApiErrorResponse` son interfaces porque nadie los instancia ni valida. Cuando se agregue Swagger, `ApiErrorResponse` pasa a clase con `@ApiProperty`; `MessageResponse<T>` necesita además un decorador helper (`ApiExtraModels` + `getSchemaPath`), porque Swagger no entiende genéricos de clase.

### Configuración

- Variable nueva: agregarla a `EnvironmentVariables`, con tipo **explícito** aunque tenga valor por defecto (`PORT: number = 3033`), y a `.env.template`.
- Leer con `config.get('VAR', { infer: true })` usando `ConfigService<EnvironmentVariables, true>`.
- Configuración de un módulo externo: `forRootAsync({ inject: [ConfigService], useFactory })` con la factory en `src/config/`.

### Base de datos

- Opciones del pool de `pg`: `poolSize` (se traduce a `max`) y `extra` con opciones de node-postgres. Nunca opciones de MySQL (`connectionLimit`, `timezone`).
- Una sola tabla: repositorio de TypeORM inyectado. Varias escrituras que deben ir juntas: `dataSource.transaction(...)`.
- Evitar N+1: `relations` o `QueryBuilder` con `leftJoinAndSelect`.

## Checklist para un módulo nuevo

1. `nest g resource <feature>` (o copiar la plantilla) y mover los archivos a `controllers/`, `services/`, `dto/`, `entities/`.
2. Crear `constants/<feature>.exception-response.ts` con al menos el mensaje `NOT_FOUND`, y `constants/<feature>.messages.ts` con un mensaje por acción.
3. DTOs con `class-validator`; `UpdateDto extends PartialType(CreateDto)` desde `@nestjs/mapped-types`.
4. Servicio que lanza `DomainException(HttpStatus.X, <Feature>ExceptionResponse.Y)` y devuelve entidades.
5. Controlador sin try/catch ni formato manual: `@ResponseMessage` en cada acción, `@HttpCode(HttpStatus.OK)` en cada consulta por `POST`.
6. `TypeOrmModule.forFeature([Entity])` en el módulo; exportar el servicio solo si otro módulo lo usa.
7. Registrar el módulo en `app.module.ts`.
8. `pnpm exec tsc --noEmit`, `pnpm run lint` y `pnpm run build` sin errores.

## Reglas relacionadas de `nestjs-best-practices`

`arch-feature-modules`, `arch-avoid-circular-deps`, `arch-module-sharing`, `error-use-exception-filters`, `error-throw-http-exceptions`, `api-use-interceptors`, `api-use-dto-serialization`, `api-use-pipes`, `security-validate-all-input`, `devops-use-config-module`, `db-avoid-n-plus-one`, `db-use-transactions`.
