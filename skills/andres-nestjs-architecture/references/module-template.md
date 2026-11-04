# Module template (example: `company`)

A full module following the base architecture: `getParams`, a paginated `getAll`, `create`, `update/:id`, a non-CRUD action and `delete/:id`. Replace `company` / `Company` with the domain name.

## `company/constants/company.exception-response.ts`

```ts
export const CompanyExceptionResponse = {
  NOT_FOUND: 'Company not found',
  // A text with values is a function in the catalog, never inline in the service
  IDENTIFICATION_TAKEN: (identification: string) =>
    `A company with identification ${identification} already exists`,
  ALREADY_INACTIVE: 'The company is already inactive',
} as const;
```

## `company/constants/company.messages.ts`

```ts
export const CompanyMessages = {
  CREATED: 'Company created successfully',
  UPDATED: 'Company updated successfully',
  DEACTIVATED: 'Company deactivated successfully',
  DELETED: 'Company deleted successfully',
} as const;
```

## `company/enums/company-type.enum.ts`

```ts
import type { EnumOptionDto } from '../../common/dto/enum-option.dto.js';

export enum CompanyTypeEnum {
  Client = 1,
  Supplier = 2,
}

export class CompanyTypeEnumHelper {
  private static readonly texts: Record<CompanyTypeEnum, string> = {
    [CompanyTypeEnum.Client]: 'Client',
    [CompanyTypeEnum.Supplier]: 'Supplier',
  };

  public static GetText(value: CompanyTypeEnum): string {
    return this.texts[value];
  }

  public static GetList(): EnumOptionDto[] {
    return Object.entries(this.texts).map(([id, value]) => ({
      id: Number(id),
      value,
    }));
  }
}
```

## `company/entities/company.entity.ts`

```ts
import { ApiProperty } from '@nestjs/swagger';
import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';

import { CompanyTypeEnum } from '../enums/company-type.enum.js';

@Entity('company')
export class Company {
  @ApiProperty({ format: 'uuid' })
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @ApiProperty({ example: '900123456' })
  @Column({ type: 'varchar', unique: true })
  identification!: string;

  @ApiProperty({ example: 'Example Company Inc.' })
  @Column({ type: 'varchar' })
  name!: string;

  @ApiProperty({ enum: CompanyTypeEnum })
  @Column({ type: 'int' })
  type!: CompanyTypeEnum;

  @ApiProperty({ example: true })
  @Column({ type: 'boolean', default: true })
  isActive!: boolean;
}
```

## `company/dto/create-company.dto.ts` and `update-company.dto.ts`

```ts
import { ApiProperty } from '@nestjs/swagger';
import { IsEnum, IsNotEmpty } from 'class-validator';

import { SafeText } from '../../common/decorators/safe-text.decorator.js';
import { CompanyTypeEnum } from '../enums/company-type.enum.js';

export class CreateCompanyDto {
  @ApiProperty({ example: '900123456', maxLength: 20 })
  @SafeText(20)
  @IsNotEmpty()
  identification!: string;

  @ApiProperty({ example: 'Example Company Inc.', maxLength: 150 })
  @SafeText(150)
  @IsNotEmpty()
  name!: string;

  @ApiProperty({ enum: CompanyTypeEnum, example: CompanyTypeEnum.Client })
  @IsEnum(CompanyTypeEnum)
  type!: CompanyTypeEnum;
}
```

```ts
// From @nestjs/swagger, not @nestjs/mapped-types: it also copies the Swagger metadata
import { ApiPropertyOptional, PartialType } from '@nestjs/swagger';
import { IsBoolean } from 'class-validator';

import { IsOptionalNotNull } from '../../common/decorators/is-optional-not-null.decorator.js';
import { CreateCompanyDto } from './create-company.dto.js';

// skipNullProperties: false → only an omitted field is skipped; a null gets a 400
export class UpdateCompanyDto extends PartialType(CreateCompanyDto, {
  skipNullProperties: false,
}) {
  @ApiPropertyOptional({ example: false })
  @IsOptionalNotNull()
  @IsBoolean()
  isActive?: boolean;
}
```

## `company/dto/company-filter.dto.ts`

```ts
import { ApiPropertyOptional } from '@nestjs/swagger';
import { Type } from 'class-transformer';
import { IsBoolean, IsEnum, IsOptional, ValidateNested } from 'class-validator';

import { BaseFilterDto } from '../../common/dto/base-filter.dto.js';
import { PagedDto } from '../../common/dto/paged.dto.js';
import { CompanyTypeEnum } from '../enums/company-type.enum.js';

// textFilter (with @SafeText) comes from BaseFilterDto: do not redeclare it
export class CompanyFilterDto extends BaseFilterDto {
  @ApiPropertyOptional({ enum: CompanyTypeEnum })
  @IsOptional()
  @IsEnum(CompanyTypeEnum)
  type?: CompanyTypeEnum;

  @ApiPropertyOptional()
  @IsOptional()
  @IsBoolean()
  isActive?: boolean;
}

// filter is redeclared with its own @Type: class-transformer cannot resolve a generic
export class CompanyPagedDto extends PagedDto<CompanyFilterDto> {
  @ApiPropertyOptional({ type: CompanyFilterDto })
  @IsOptional()
  @ValidateNested()
  @Type(() => CompanyFilterDto)
  declare filter?: CompanyFilterDto;
}
```

## `company/dto/company-params-response.dto.ts`

```ts
import { ApiProperty } from '@nestjs/swagger';

import { EnumOptionDto } from '../../common/dto/enum-option.dto.js';
import { CompanyTypeEnumHelper } from '../enums/company-type.enum.js';

export class CompanyParamsResponseDto {
  // The helper's own list, so the docs never drift from the API
  @ApiProperty({ type: [EnumOptionDto], example: CompanyTypeEnumHelper.GetList() })
  types!: EnumOptionDto[];
}
```

## `company/services/company.service.ts`

```ts
import { HttpStatus, Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Not, Repository } from 'typeorm';

import { DomainException } from '../../common/exceptions/domain.exception.js';
import { escapeLike } from '../../common/helpers/escape-like.helper.js';
import type { PaginatedResult } from '../../common/interfaces/paginated-result.interface.js';
import { CompanyExceptionResponse } from '../constants/company.exception-response.js';
import { CompanyPagedDto } from '../dto/company-filter.dto.js';
import { CompanyParamsResponseDto } from '../dto/company-params-response.dto.js';
import { CreateCompanyDto } from '../dto/create-company.dto.js';
import { UpdateCompanyDto } from '../dto/update-company.dto.js';
import { Company } from '../entities/company.entity.js';
import { CompanyTypeEnumHelper } from '../enums/company-type.enum.js';

@Injectable()
export class CompanyService {
  constructor(
    @InjectRepository(Company)
    private readonly companyRepository: Repository<Company>,
  ) {}

  getParams(): CompanyParamsResponseDto {
    return { types: CompanyTypeEnumHelper.GetList() };
  }

  // No getById: a row of the list already carries everything the edit form needs
  async getAll(pagedDto: CompanyPagedDto): Promise<PaginatedResult<Company>> {
    const { page, size } = pagedDto.pagination;
    const { textFilter, type, isActive } = pagedDto.filter ?? {};

    const queryBuilder = this.companyRepository.createQueryBuilder('company');

    if (textFilter) {
      queryBuilder.andWhere(
        '(company.name ILIKE :textFilter OR company.identification ILIKE :textFilter)',
        { textFilter: `%${escapeLike(textFilter)}%` },
      );
    }

    if (type != null) {
      queryBuilder.andWhere('company.type = :type', { type });
    }

    if (isActive != null) {
      queryBuilder.andWhere('company.isActive = :isActive', { isActive });
    }

    const [rows, count] = await queryBuilder
      .orderBy('company.name', 'ASC')
      .skip((page - 1) * size)
      .take(size)
      .getManyAndCount();

    return { page, size, count, rows };
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
    if (dto.identification) {
      await this.ensureIdentificationIsFree(dto.identification, id);
    }
    return this.companyRepository.save(company);
  }

  async deactivate(id: string): Promise<Company> {
    const company = await this.companyRepository.findOneBy({ id });
    if (!company) {
      throw new DomainException(HttpStatus.NOT_FOUND, CompanyExceptionResponse.NOT_FOUND);
    }
    if (!company.isActive) {
      throw new DomainException(HttpStatus.CONFLICT, CompanyExceptionResponse.ALREADY_INACTIVE);
    }
    company.isActive = false;
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
  private async ensureIdentificationIsFree(
    identification: string,
    excludeId?: string,
  ): Promise<void> {
    const exists = await this.companyRepository.existsBy(
      excludeId ? { identification, id: Not(excludeId) } : { identification },
    );
    if (exists) {
      throw new DomainException(
        HttpStatus.CONFLICT,
        CompanyExceptionResponse.IDENTIFICATION_TAKEN(identification),
      );
    }
  }
}
```

## `company/controllers/company.controller.ts`

When the API has authentication, every handler also carries its own `@Auth(...)` right after the route decorator and its `@HttpCode` (see Authorization in the SKILL).

```ts
import {
  Body,
  Controller,
  Delete,
  Get,
  HttpCode,
  HttpStatus,
  Param,
  ParseUUIDPipe,
  Post,
  Put,
} from '@nestjs/common';
import { ApiOkResponse, ApiOperation, ApiTags } from '@nestjs/swagger';

import { ApiPagedBody } from '../../common/decorators/api-paged-body.decorator.js';
import {
  ApiErrorResponses,
  ApiMessageResponse,
  ApiPaginatedResponse,
} from '../../common/decorators/api-responses.decorator.js';
import { ResponseMessage } from '../../common/decorators/response-message.decorator.js';
import { CompanyMessages } from '../constants/company.messages.js';
import { CompanyPagedDto } from '../dto/company-filter.dto.js';
import { CompanyParamsResponseDto } from '../dto/company-params-response.dto.js';
import { CreateCompanyDto } from '../dto/create-company.dto.js';
import { UpdateCompanyDto } from '../dto/update-company.dto.js';
import { Company } from '../entities/company.entity.js';
import { CompanyTypeEnum } from '../enums/company-type.enum.js';
import { CompanyService } from '../services/company.service.js';

@ApiTags('Companies')
@Controller('company')
export class CompanyController {
  constructor(private readonly companyService: CompanyService) {}

  // Queries: no @ResponseMessage, the result is returned as is

  @Get('getParams')
  @ApiOperation({ summary: 'Get the lists the company forms need' })
  @ApiOkResponse({ type: CompanyParamsResponseDto })
  getParams() {
    return this.companyService.getParams();
  }

  // Query over POST: 200 instead of Nest's default 201
  @Post('getAll')
  @HttpCode(HttpStatus.OK)
  @ApiOperation({
    summary: 'Get all companies, paginated and filtered',
    description: '`textFilter` is a partial match on the name or the identification.',
  })
  @ApiPagedBody(CompanyPagedDto, {
    allFilters: {
      summary: 'Every filter (remove the ones you do not need)',
      filter: { textFilter: 'Example', type: CompanyTypeEnum.Client, isActive: true },
    },
    byIdentification: { summary: 'By identification', filter: { textFilter: '900123456' } },
  })
  @ApiPaginatedResponse(Company)
  @ApiErrorResponses(HttpStatus.BAD_REQUEST)
  getAll(@Body() pagedDto: CompanyPagedDto) {
    return this.companyService.getAll(pagedDto);
  }

  // Actions: always @ResponseMessage, the response is { message, data? }

  @Post('create')
  @ResponseMessage(CompanyMessages.CREATED)
  @ApiOperation({ summary: 'Create a company', description: '`identification` is unique.' })
  @ApiMessageResponse(HttpStatus.CREATED, Company)
  @ApiErrorResponses(HttpStatus.BAD_REQUEST, HttpStatus.CONFLICT)
  create(@Body() dto: CreateCompanyDto) {
    return this.companyService.create(dto);
  }

  @Put('update/:id')
  @ResponseMessage(CompanyMessages.UPDATED)
  @ApiOperation({ summary: 'Update a company' })
  @ApiMessageResponse(HttpStatus.OK, Company)
  @ApiErrorResponses(HttpStatus.BAD_REQUEST, HttpStatus.NOT_FOUND, HttpStatus.CONFLICT)
  update(@Param('id', ParseUUIDPipe) id: string, @Body() dto: UpdateCompanyDto) {
    return this.companyService.update(id, dto);
  }

  // Non-CRUD action: same pattern
  @Put('deactivate/:id')
  @ResponseMessage(CompanyMessages.DEACTIVATED)
  @ApiOperation({ summary: 'Deactivate a company' })
  @ApiMessageResponse(HttpStatus.OK, Company)
  @ApiErrorResponses(HttpStatus.NOT_FOUND, HttpStatus.CONFLICT)
  deactivate(@Param('id', ParseUUIDPipe) id: string) {
    return this.companyService.deactivate(id);
  }

  // 200 (Nest's default) so the message reaches the client; a 204 has no body
  @Delete('delete/:id')
  @ResponseMessage(CompanyMessages.DELETED)
  @ApiOperation({ summary: 'Delete a company' })
  @ApiMessageResponse(HttpStatus.OK)
  @ApiErrorResponses(HttpStatus.NOT_FOUND, HttpStatus.CONFLICT)
  remove(@Param('id', ParseUUIDPipe) id: string) {
    return this.companyService.remove(id);
  }
}
```

## Batch actions

The message is fixed and the summary goes in `data`; the service returns numbers, not a text.

```ts
// company/constants/company.messages.ts
export const CompanyMessages = {
  // ...
  IMPORTED: 'Companies imported successfully',
} as const;
```

```ts
// → { message, data: { created: 3, skipped: 2 } }
@Post('import')
@ResponseMessage(CompanyMessages.IMPORTED)
import(@Body() dto: ImportCompaniesDto) {
  return this.companyService.import(dto);
}
```

```ts
async import(dto: ImportCompaniesDto): Promise<{ created: number; skipped: number }> {
  let created = 0;
  let skipped = 0;

  for (const row of dto.companies) {
    // "already exists" is an expected result of the process: it is counted, not thrown
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

## Resulting responses

`GET /api/v1/company/getParams` → `200` (query: the result as is)
```json
{ "types": [{ "id": 1, "value": "Client" }, { "id": 2, "value": "Supplier" }] }
```

`POST /api/v1/company/getAll` → `200` (query over `POST`: the result as is)
```json
{
  "page": 1,
  "size": 10,
  "count": 42,
  "rows": [{ "id": "7c0e...", "identification": "900123456", "name": "Example Company Inc.", "type": 1, "isActive": true }]
}
```

`POST /api/v1/company/create` → `201` (action that returns the entity)
```json
{
  "message": "Company created successfully",
  "data": { "id": "7c0e...", "identification": "900123456", "name": "Example Company Inc.", "type": 1, "isActive": true }
}
```

`DELETE /api/v1/company/delete/7c0e...` → `200` (action with no return value: only the message)
```json
{ "message": "Company deleted successfully" }
```

`PUT /api/v1/company/update/<nonexistent uuid>` → `404`
```json
{
  "statusCode": 404,
  "message": "Company not found",
  "timestamp": "2026-09-15T15:00:00.000Z",
  "path": "/api/v1/company/update/..."
}
```

`PUT /api/v1/company/update/7c0e...` with `{ "name": null }` → `400` (`skipNullProperties: false`)
```json
{
  "statusCode": 400,
  "message": ["name should not be empty", "name must be a string", "..."],
  "timestamp": "2026-09-15T15:00:00.000Z",
  "path": "/api/v1/company/update/..."
}
```
