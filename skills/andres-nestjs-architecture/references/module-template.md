# Plantilla de módulo (ejemplo: `company`)

CRUD completo siguiendo la arquitectura base. Reemplazar `company` / `Company` por el nombre del dominio.

## `company/constants/company.exception-response.ts`

```ts
export const CompanyExceptionResponse = {
  NOT_FOUND: 'Company not found',
  IDENTIFICATION_TAKEN: 'A company with this identification already exists',
} as const;
```

## `company/constants/company.messages.ts`

```ts
export const CompanyMessages = {
  CREATED: 'Company created successfully',
  UPDATED: 'Company updated successfully',
  DELETED: 'Company deleted successfully',
} as const;
```

## `company/entities/company.entity.ts`

```ts
import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';

@Entity('company')
export class Company {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Column({ type: 'varchar', unique: true })
  identification!: string;

  @Column({ type: 'varchar' })
  name!: string;
}
```

## `company/dto/create-company.dto.ts` y `update-company.dto.ts`

```ts
import { IsNotEmpty, IsString } from 'class-validator';

export class CreateCompanyDto {
  @IsString()
  @IsNotEmpty()
  identification!: string;

  @IsString()
  @IsNotEmpty()
  name!: string;
}
```

```ts
import { PartialType } from '@nestjs/mapped-types';

import { CreateCompanyDto } from './create-company.dto.js';

export class UpdateCompanyDto extends PartialType(CreateCompanyDto) {}
```

## `company/services/company.service.ts`

```ts
import { HttpStatus, Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';

import { DomainException } from '../../common/exceptions/domain.exception.js';
import { CompanyExceptionResponse } from '../constants/company.exception-response.js';
import { CreateCompanyDto } from '../dto/create-company.dto.js';
import { UpdateCompanyDto } from '../dto/update-company.dto.js';
import { Company } from '../entities/company.entity.js';

@Injectable()
export class CompanyService {
  constructor(
    @InjectRepository(Company)
    private readonly companyRepository: Repository<Company>,
  ) {}

  findAll(): Promise<Company[]> {
    return this.companyRepository.find({ order: { name: 'ASC' } });
  }

  async findOne(id: string): Promise<Company> {
    const company = await this.companyRepository.findOneBy({ id });
    if (!company) {
      throw new DomainException(HttpStatus.NOT_FOUND, CompanyExceptionResponse.NOT_FOUND);
    }
    return company;
  }

  async create(dto: CreateCompanyDto): Promise<Company> {
    await this.ensureIdentificationIsFree(dto.identification);
    return this.companyRepository.save(this.companyRepository.create(dto));
  }

  async update(id: string, dto: UpdateCompanyDto): Promise<Company> {
    const company = await this.companyRepository.preload({ id, ...dto });
    if (!company) {
      throw new DomainException(HttpStatus.NOT_FOUND, CompanyExceptionResponse.NOT_FOUND);
    }
    return this.companyRepository.save(company);
  }

  async remove(id: string): Promise<void> {
    const { affected } = await this.companyRepository.delete(id);
    if (!affected) {
      throw new DomainException(HttpStatus.NOT_FOUND, CompanyExceptionResponse.NOT_FOUND);
    }
  }

  // Optional: the unique index already yields a generic 409 (UNIQUE_VIOLATION);
  // checking first gives the client a module-specific message.
  private async ensureIdentificationIsFree(identification: string): Promise<void> {
    const exists = await this.companyRepository.existsBy({ identification });
    if (exists) {
      throw new DomainException(
        HttpStatus.CONFLICT,
        CompanyExceptionResponse.IDENTIFICATION_TAKEN,
      );
    }
  }
}
```

## `company/controllers/company.controller.ts`

```ts
import {
  Body,
  Controller,
  Delete,
  Get,
  Param,
  ParseUUIDPipe,
  Patch,
  Post,
} from '@nestjs/common';

import { ResponseMessage } from '../../common/decorators/response-message.decorator.js';
import { CompanyMessages } from '../constants/company.messages.js';
import { CreateCompanyDto } from '../dto/create-company.dto.js';
import { UpdateCompanyDto } from '../dto/update-company.dto.js';
import { CompanyService } from '../services/company.service.js';

@Controller('companies')
export class CompanyController {
  constructor(private readonly companyService: CompanyService) {}

  // Queries: no @ResponseMessage, the result is returned as-is

  @Get()
  findAll() {
    return this.companyService.findAll();
  }

  @Get(':id')
  findOne(@Param('id', ParseUUIDPipe) id: string) {
    return this.companyService.findOne(id);
  }

  // Actions: always @ResponseMessage, the response is { message, data? }

  @Post()
  @ResponseMessage(CompanyMessages.CREATED)
  create(@Body() dto: CreateCompanyDto) {
    return this.companyService.create(dto);
  }

  @Patch(':id')
  @ResponseMessage(CompanyMessages.UPDATED)
  update(@Param('id', ParseUUIDPipe) id: string, @Body() dto: UpdateCompanyDto) {
    return this.companyService.update(id, dto);
  }

  // 200 (Nest default) so the message reaches the client; a 204 has no body
  @Delete(':id')
  @ResponseMessage(CompanyMessages.DELETED)
  remove(@Param('id', ParseUUIDPipe) id: string) {
    return this.companyService.remove(id);
  }
}
```

## Consultas con paginación o filtros en el body (`POST`)

Cuando los filtros no caben cómodamente en query params, la consulta va por `POST`. Sigue siendo una consulta: **sin `@ResponseMessage`** y **con `@HttpCode(HttpStatus.OK)`**, porque Nest responde `201` por defecto en todo `POST`.

```ts
import { HttpCode, HttpStatus, Post, Body } from '@nestjs/common';

@Post('search')
@HttpCode(HttpStatus.OK)
search(@Body() pagedDto: CompanyPagedDto) {
  return this.companyService.search(pagedDto); // { page, size, count, rows }
}
```

> Los DTOs de paginación comunes (`PaginationDto`, filtro base y resultado paginado tipado) están pendientes: ver `docs/pendientes-estructura-base.md`.

## Acciones que no son CRUD

Mismo patrón: el servicio devuelve datos (o nada) y el controlador declara el mensaje.

```ts
// company/constants/company.messages.ts
export const CompanyMessages = {
  // ...
  ACTIVATED: 'Company activated successfully',
  IMPORTED: 'Companies imported successfully',
} as const;
```

```ts
// Cambio de estado: devuelve la entidad actualizada → { message, data }
@Patch(':id/activate')
@ResponseMessage(CompanyMessages.ACTIVATED)
activate(@Param('id', ParseUUIDPipe) id: string) {
  return this.companyService.activate(id);
}

// Proceso por lotes: el mensaje es fijo y el detalle va en data
// → { message, data: { created: 3, skipped: 2 } }
@Post('import')
@ResponseMessage(CompanyMessages.IMPORTED)
import(@Body() dto: ImportCompaniesDto) {
  return this.companyService.import(dto);
}
```

```ts
// El servicio devuelve el resumen, no un texto
async import(dto: ImportCompaniesDto): Promise<{ created: number; skipped: number }> {
  let created = 0;
  let skipped = 0;

  for (const row of dto.companies) {
    // "ya existe" es un resultado esperado del proceso: se cuenta, no se lanza
    if (await this.companyRepository.existsBy({ identification: row.identification })) {
      skipped += 1;
      continue;
    }
    await this.companyRepository.save(this.companyRepository.create(row));
    created += 1;
  }

  return { created, skipped };
}
```

> El decorador va en el handler, no en el `@Controller`: a nivel de clase el mensaje se aplicaría también a las consultas.

## `company/company.module.ts`

```ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

import { CompanyController } from './controllers/company.controller.js';
import { Company } from './entities/company.entity.js';
import { CompanyService } from './services/company.service.js';

@Module({
  imports: [TypeOrmModule.forFeature([Company])],
  controllers: [CompanyController],
  providers: [CompanyService],
  // exports: [CompanyService], // only if another module needs it
})
export class CompanyModule {}
```

## Respuestas resultantes

`GET /api/v1/companies/7c0e...` → `200` (consulta: el resultado tal cual)
```json
{ "id": "7c0e...", "identification": "900123456", "name": "Acme S.A." }
```

`POST /api/v1/companies/search` → `200` (consulta por `POST`: el resultado tal cual)
```json
{ "page": 1, "size": 10, "count": 42, "rows": [{ "id": "7c0e...", "name": "Acme S.A." }] }
```

`POST /api/v1/companies` → `201` (acción que devuelve la entidad)
```json
{
  "message": "Company created successfully",
  "data": { "id": "7c0e...", "identification": "900123456", "name": "Acme S.A." }
}
```

`DELETE /api/v1/companies/7c0e...` → `200` (acción sin retorno: solo el mensaje)
```json
{ "message": "Company deleted successfully" }
```

`GET /api/v1/companies/<uuid inexistente>` → `404`
```json
{
  "statusCode": 404,
  "message": "Company not found",
  "timestamp": "2026-09-15T15:00:00.000Z",
  "path": "/api/v1/companies/..."
}
```

`POST /api/v1/companies` con body vacío → `400`
```json
{
  "statusCode": 400,
  "message": ["identification should not be empty", "identification must be a string", "..."],
  "timestamp": "2026-09-15T15:00:00.000Z",
  "path": "/api/v1/companies"
}
```
