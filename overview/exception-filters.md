# Exception Filters

## Throwing Standard Exceptions:

```typescript
@Get()
async findAll() {
  throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
}
```

> `HttpStatus` is imported from `@nestjs/common` package

an example overriding the entire response body -

```typescript
@Get()
async findAll() {
  throw new HttpException({
    status: HttpStatus.FORBIDDEN,
    error: 'This is a custom message',
  }, HttpStatus.FORBIDDEN);
}
```

## Custom Exceptions:

```typescript
export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```

Since `ForbiddenException` extends the base `HttpException`, it will work seamlessly with the built-in exception handler, and therefore can be used inside the `findAll()` method.

```typescript
@Get()
async findAll() {
  throw new ForbiddenException();
}
```

## Built-in HTTP Exceptions:

- **BadRequestException**
- **UnauthorizedException**
- **NotFoundException**
- **ForbiddenException**
- **NotAcceptableException**
- **RequestTimeoutException**
- **ConflictException**
- **GoneException**
- **HttpVersionNotSupportedException**
- **PayloadTooLargeException**
- **UnsupportedMediaTypeException**
- **UnprocessableExntityException**
- **InternalServerErrorException**
- **NotImplementedException**
- **ImATeapotException**
- **MethodNotAllowedException**
- **BadGatewayException**
- **ServiceUnavailableException**
- **GatewayTimeoutException**
- **PreconditionalFailedException**

## Exception Filters:

An exception filter that is responsible for catchin exceptions which are an instance of the `HttpException` class.

```typescript
import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpException,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}
```

## Binding Filters:

```typescript
@Post()
@UseFilters(new HttpExceptionFilter())
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

`@UseFilters()` decorator is imported from `@nestjs/common` package. It can take a single filter instance, or a comma-separated list of filter instances.
Alternatively, it can take the class (instead of an instance), leaving responsibility for instantiation to the framework, and enabling **dependency inejction**.

```typescript
@Post()
@UseFilters(HttpExceptionFilter)
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

Exception filters can be scoped at different levels: method-scoped, controller-scoped, or global-scoped.

### `controller-scoped` exception filter:

```typescript
@UseFilters(new HttpExceptionFilter())
export class CatsController {}
```

### `global-scoped` exception filter:

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(3000);
}
bootstrap();
```

> The `useGlobalFilters()` method does not set up filters for _gateways_ or _hybrid applications_.

register _global-scoped_ filters directly from any module -

```typescript
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

## Catch Everything:

```typescript
import { ArgumentsHost, Catch, ExceptionFilter } from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';

@Catch()
export class AllExceptionFilter implements ExceptionFilter {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    // In certain situations `httpAdapter` might not be available in the
    // contructor method, thus we should resolve it here
    const { httpAdapter } = this.httpAdapterHost;

    const ctx = host.switchToHttp();

    const httpStatus = exception instanceOf HttpException ? exception.getStatus() : HttpStatus.INTERNAL_SERVER_ERROR;

    const responseBody = {
      statusCode: httpStatus,
      timestamp: new Date().toISOString(),
      path: httpAdapter.getRequestUrl(ctx.getRequest()),
    };

    httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
  }
}
```

## Inheritance:

In order to delegate exception processing to the base filter, it is needed to extend `BaseExceptionFilter` and call the inherited `catch()` method.

```typescript
import { ArgumentsHost, Catch } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    super.catch(exception, host);
  }
}
```

_Method-scoped_ and _Controller-scoped_ filters that extend the `BaseExceptionFilter` should not be instantiated with _new_. Instead, let the framework instantiate them autometically.

Global filters can extend the base filter. This can be done in either of two ways.

### `#1`:

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const { httpAdapter } = app.get(HttpAdapterHost);
  app.useGlobalFilters(new AllExceptionFilter(httpAdapter));

  await app.listen(3000:)
}
```

### `#2`:

```typescript
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: AllExceptionFilter,
    },
  ],
})
export class AppModule {}
```
