# Middleware

Middleware functions can perform the following tasks -

- execute any code.
- make changes to the request and the response objects.
- end the request-response cycle.
- call the next middlware function in the stack.
- if the current middleware function does not end the request-response cycle, it must call `next()` to pass control to the next middleware function. Otherwise, the request will be left hangin

## Implementing Middleware:

```typescript
import { Injectable, NestMiddleware } from '@nestjs/common';
import { NextFunction, Request, Response } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log('Request....');
    next();
  }
}
```

## Applying Middleware:

### `#1`:

```typescript
// app.module.ts
import { MiddlewareConsumer, Module, NestModule } from '@nestjs/common';
import { LoggerMiddleware } from './middlewares/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes('cats');
  }
}
```

### `#2`:

```typescript
// app.module.ts
import { MiddlewareConsumer, Module, NestModule } from '@nestjs/common';
import { LoggerMiddleware } from './middlewares/logger.middleware';
import { CatsController } from './cats/cats.controller';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes(CatsController);
  }
}
```

### `#3`:

```typescript
// app.module.ts
import {
  MiddlewareConsumer,
  Module,
  NestModule,
  RequestMethod,
} from '@nestjs/common';
import { LoggerMiddleware } from './middlewares/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
```

Route wildcars -

```typescript
forRoutes({ path: 'ab*cd', method: RequestMethod.ALL });
```

## Excluding routes:

```typescript
consumer
  .apply(LoggerMiddlware)
  .exclude(
    { path: 'cats', method: RequestMethod.GET },
    { path: 'cats', method: RequestMethod.POST },
    'cats/(.*)',
  )
  .forRoutes(CatsController);
```

> The `exclude()` method supports wildcar parameters using the [path-to-regexp](https://github.com/pillarjs/path-to-regexp#parameters) package.

## Functional Middleware:

```typescript
import { NextFunction, Request, Response } from 'express';

export function logger(req: Request, res: Response, next: NextFunction) {
  console.log('Request.....');
  next();
}
```

And use it within the `AppModule`:

```typescript
consumer.apply(logger).forRoutes(CatsController);
```

## Multiple Middleware:

```typescript
consumer.apply(cors(), helmet(), logger).forRoutes(CatsController);
```

## Global Middleware:

```typescript
const app = await NestFactory.create(AppModule);
app.use(logger);
await app.listen(3000);
```
