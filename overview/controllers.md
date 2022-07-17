# Controllers

## `main.ts`:

```typescript
import { NestFactory } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
```

By default, if any error happens while creating the application app will exit with the code 1.
To make it throw an error instead disable the option `abortOnError`

```typescript
const app = await NestFactory.create(AppModule, { abortOnError: false });
```

## Routing:

### `cats.controller.ts`:

```typescript
import {
  Body,
  Controller,
  Delete,
  Get,
  Header,
  HttpCode,
  HttpStatus,
  Param,
  Patch,
  Post,
  Query,
} from '@nestjs/common';
import { Request, Response } from 'express';
import { CreateCatDto } from './dtos/create-cat.dto.ts';
import { QueryCatDto } from './dtos/query-cat.dto.ts';
import { UpdateCatDto } from './dtos/update-cat.dto.ts';

@Controller('cats')
export class CatsController {
  @Get()
  @HttpCode(HttpStatus.FOUND)
  async findAll(@Req() request: Request, @Query() query: QueryCatDto) {
    return `This action returns all cats (limit: ${query.limit} items).`;
  }

  @Post()
  @Header('Cache-Control', 'none')
  async create(
    @Res({ passthrough: true }) response: Response,
    @Body() createCatDto: CreateCatDto,
  ) {
    return `This action adds a new cat.`;
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return `This action returns a #${id} cat.`;
  }

  @Patch(':id')
  @HttpCode(HttpStatus.OK)
  update(@Param('id') id: string, @Body() body: UpdateCatDto) {
    return `This action updates a #${id} cat.`;
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return `This action removes a #${id} cat.`;
  }
}
```

### `create-cat.dto.ts`:

```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

### `update-cat.dto.ts`:

```typescript
import { PartialType } from '@nestjs/mapped-types';
import { CreateCatDto } from './create-cat.dto.ts';

export class UpdateCatDto extends PartialType(CreateCatDto) {}
```

### `query-cat.dto.ts`:

```typescript
import { PartialType } from '@nestjs/mapped-types';
import { CreateCatDto } from './create-cat.dto.ts';

export class QueryCatDto extends PartialType(CreateCatDto) {}
```

## List of Decorators and the plain platform-specific objects they represent:

| Decorator               | Platform-specific Object        |
| ----------------------- | ------------------------------- |
| @Request(), @Req()      | req                             |
| @Response(), @Res()     | res                             |
| @Next()                 | next                            |
| @Session()              | req.session                     |
| @Param(key?: string)    | req.params / req.params[key     |
| @Body(key?: string)     | req.body / req.body[key]        |
| @Query(key?: string)    | req.query / req.query[key]      |
| @Headers(name?: string) | req.headers / req.headers[name] |
| @Ip()                   | req.ip                          |
| @HostParam()            | req.hosts                       |

## Decorators for Http methods:

| Decorator  | Http Method |
| ---------- | ----------- |
| @Get()     | GET         |
| @Post()    | POST        |
| @Put()     | PUT         |
| @Delete()  | DELETE      |
| @Patch()   | PATCH       |
| @Options() | OPTIONS     |
| @Head()    | HEAD        |
| @All()     | ALL         |

## Route Wildcars:

> The characters `?`, `+`, `*`, and `()` may be used in a route path

```typescript
@Get('ab*cd')
findAll() {
  return 'This route uses a wildcard';
}
```

The `ab*cd` route path will match `abcd`, `ab_cd`, `abecd`, and so on.

## Status Code:

```typescript
@Get()
@HttpCode(HttpStatus.NO_CONTENT) // HttpStatus.NO_CONTENT === 204
create() {
  return 'This action adds a new cat';
}
```

`HttpCode` and `HttpStatus` is imported from `@nestjs/common` package.

## Headers:

```typescript
@Get()
@Header('Cache-Control', 'none')
create() {
  return 'This action adds a new cat.';
}
```

`Header` is imported from `@nestjs/common`.

## Redirection:

`@Redirect()` takes two arguments, `url`, and `statusCode`, both are optional.
The default value of `statusCode` is `302` (`Found`) if ommited.

```typescript
@Get()
@Redirect('https://nestjs.com', 301)
```

To determine the `statusCode` or `url` dynamically return an object from the route handle with the shape

```typescript
{
  "url": string,
  "statusCode": number
}
```

Returned values will override any arguments passed to the `@Redirect()` decorator.

```typescript
@Get()
@Redirect('https://docs.nestjs.com', 302)
getDocs(@Query('version') version) {
  if(version && version === '5') {
    return { url: 'https://docs.nestjs.com/v5' };
  }
}
```

## Sub-domain Routing:

The controller decorator can take a `host` option to require that the HTTP host of the incomming request matches specific value.

```typescript
@Controller({ host: 'admin.example.com' })
export class AdminController {
  @Get()
  index(): string {
    return 'Admin page';
  }
}
```

Similart to a `route` path, the `hosts` option can use tokens to capture the dynamic value at that position in the host name.

```typescript
@Controller({ host: ':account.example.com' })
export class AccountController {
  @Get()
  getInfo(@HostParam('account') account): string {
    return account;
  }
}
```
