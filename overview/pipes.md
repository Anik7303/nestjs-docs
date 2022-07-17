# Pipes:

A pipe is a class annotated with the `@Injectable()` decorator, which implements the `PipeTransform` interface.

## Built-in Pipes:

- ValidationPipe
- ParseIntPipe
- ParseFloatPipe
- ParseBoolPipe
- ParseArrayPipe
- ParseUUIDPipe
- ParseEnumPipe
- DefaultValuePipe
- ParseFilePipe

They are exported from `@nestjs/common` package.

## Binding Pipes:

```typescript
@Get('id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

Passing an in-place instance is useful if we want to customize the built-in pipe's behavior by passing options -

```typescript
@Get('id')
async findOne(
  @Param('id', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE }))
  id: number
) {
  return this.catsService.findOne(id);
}

```

Binding the other transformation pipes (ParseFloatPipe, ParseBoolPipe, ParseFilePipe, ParseArrayPipe, ParseUUIDPipe & ParseEnumPipe) works similarly. These pipes all work in the context of validating route parameters, query string parameters and request body values.

> An example with a query string parameter

```typescript
@Get()
async findOne(@Query('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

> An example of using `ParseUUIDPipe` to parse a string parameter and validate if it is a UUID.

```typescript
@Get(':uuid')
async findOne(@Param('uuid', new ParseUUIDPipe()) uuid: string) {
  return this.catsService.findOne(uuid);
}
```

## Custom Pipes:

```typescript
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
```

`PipeTransform<T, R>` is a generic interface that must be implemented by any pipe. The generic interface uses `T` to indicate the type of the input `value`, and `R` to indicate the return type of the `transform()` method.
`PipeTransform` has two parameters -

- _value_
- _metadata_
  The _value_ parameter is the currently processed method argument and _metadata_ is the currently processed method argument's metadata.
  The metadata object has these properties -

```typescript
export interface ArgumentMetadata {
  type: 'body' | 'query' | 'param' | 'custom';
  metatype?: Type<unknown>;
  data?: string;
}
```

## Schema Based Validation:

```typescript
// cats.controller.ts
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

```typescript
// create-cat.dto.ts
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

### Object Schema Validation:

One common approach is to use `schema-based` validation.
The [Joi](https://www.npmjs.com/package/joi) library allows yo to create schemas in a straightforward way, with a readable API.

```typescript
import { ArgumentMetadata, Injectable, PipeTransform } from '@nestjs/common';
import { ObjectSchema } from 'joi';

@Injectable()
export class JoiValidationPipe implements PipeTransform {
  constructor(private schema: ObjectSchema) {}

  transform(value: any, metadata: ArgumentMetadata) {
    const { error } = this.schema.validate(value);
    if (error) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }
}
```

### Binding Validation Pipes:

steps:

- create an instance of the `JoiValidationPipe`
- pass the context-specific Joi schema in the class constructor of the pipe
- bind the pipe to the method

```typescript
@Post()
@UsePipes(new JoiValidationPipe(createCatSchema))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

> The `@UsePipes()` decorator is imported from the `@nestjs/common` package.

### Class Validator:

> This technique requires _Typescript_, and are not available if the app is written in vanilla Javascript.

```typescript
import { IsInt, IsString } from 'class-validator';

export class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  age: number;

  @IsString()
  breed: string;
}
```

```typescript
// pipes/validation.pipe.ts
import {
  ArgumentMetadata,
  BadRequestException,
  Injectable,
  PipeTransform,
} from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }

    const object = plainToInstance(metatype, value);
    const errors = await validate(object);
    if (errors.length > 0) {
      return BadRequestException('Validation failed');
    }
    return value;
  }

  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

> Bind the `ValidationPipe`
> Pipes can be parameter-scoped, method-scoped, controller-scoped, or global-scoped.

```typescript
@Post()
async create(
  @Body(new ValidationPipe()) createCatDto: CreateCatDto,
) {
  this.catsService.create(createCatDto);
}
```

## Global-scoped Pipes:

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(300);
}
bootstrap();
```

> In order to inject dependencies, set up a global pipe directly from any module -

```typescript
import { Module } from '@nestjs/common';
import { APP_PIPE } from '@nestjs/core';
import { ValidationPipe } from './pipes/validation.pipe';

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: ValidationPipe,
    },
  ],
})
export class AppModule {}
```

## Transformation use case:

```typescript
// parse-int.pipe.ts
import {
  ArgumentMetadata,
  BadRequestException,
  Injectable,
  PipeTransform,
} from '@nestjs/common';

@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNan(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
```

bind this pipe

```typescript
@Get(':id')
async findOne(@Param('id', new ParseIntPipe()) id: number) {
  return this.catsService.findOne(id);
}
```

Another useful transformation case would be to select an _existing user_ entity from the database using a id supplied in the request -

```typescript
@Get(':id')
findOne(@Param('id', UserByIdPipe) userEntity: UserEntity) {
  return userEntity;
}
```

## Providing Defaults:

> `Parse*` pipes expect a parameter's value to be defined. They throw an exception upon receiving _null_ or _undefined_ values.

```typescript
import {
  Controller,
  DefaultValuePipe,
  ParseBoolPipe,
  ParseIntPipe,
} from '@nestjs/common';
import { CatsService } from './cats.service';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Get()
  async findAll(
    @Query('activeOnly', new DefaultValuePipe(false), ParseBoolPipe)
    activeOnly: boolean,
    @Query('page', new DefaultValuePipe(0), ParseIntPipe) page: number,
  ) {
    return this.catsService.findAll({ activeOnly, page });
  }
}
```
