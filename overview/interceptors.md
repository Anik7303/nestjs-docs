# Interceptors:

An interceptor is a class annotated with the `@Injectable()` decorator and implements the `NestInterceptor` interface.
Interceptors have a set of useful capabilities which are inspired by the [Aspect Oriented Programming (AOP)](https://en.wikipedia.org/wiki/Aspect-oriented_programming) technique. They make it possible to -

- bind extra logic before / after method execution
- transform the result returned from a function
- transform the exception thrown from a function
- extend the basic function behavior
- completely override a function depending on specific conditions(e.g., for caching purpose)

## Basics:

Each interceptor implements the `intercept()` method, which takes two arguments -

- Execution Context (ExecutionContext)
- Call Handler (CallHandler)

## Aspect Interception:

An interceptor to log user interaction (e.g., storing user calls, asynchronously dispatching events or calculating a timestamp)

```typescript
// loggin.interceptor.ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from "@nestjs/common";
import { Observable } from "rxjs";
import { tap } from "rxjs/operators";

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(_context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log("Before...");

    const now = Date.now();
    return next.handle().pipe(
      tap(() => {
        console.log(`After... ${Date.now() - now}ms`);
      })
    );
  }
}
```

> The `NestInterceptor<T, R>` is a generic interface in which `T` indicates the type of an `Observable<T>`(supporting the response stream), and `R` is the type of the value wrapped by `Observable<R>`.

`handle()` returns an RxJS `Observable`, we have a wide choice of operators to manipulate the stream.
`tap()` operator invokes our anonymous logging function upon graceful or exceptional termination of the observable stream.

## Binding Interceptors:

Interceptors can be controller-scoped, method-scoped, or global-scoped.

### `#1`:

```typescript
// cats.controller.ts
// controller-scoped
@UseInterceptors(LoggingInterceptor)
@Controller("cats")
export class CatsController {}
```

> The `@UseInterceptors()` decorator is imported from `@nestjs/common` package.

### `#2`:

```typescript
// cats.controller.ts
// controller-scoped
@UseInterceptors(new LoggingInterceptor())
@Controller("cats")
export class CatsController {}
```

### `#3`:

```typescript
// main.ts
// global-scoped
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalInterceptors(new LoggingInterceptor());
  await app.listen(3000);
}
bootstrap();
```

### `#4`:

```typescript
// app.module.ts
// global-scoped
import { Module } from "@nestjs/common";
import { APP_INTERCEPTOR } from "@nestjs/core";
import { LoggingInterceptor } from "./interceptors/logging.interceptor";

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

## Response Mapping:

`handle()` returns an `Observable`. The stream contains the value **returned** from the route handler, and thus we can easily mutate it using RxJS's `map()` operator.

> The response mapping feature doesn't work with the library-specific response strategy (using the `@Res()` object directly is forbidden).

```typescript
// transform.interceptor.ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from "@nestjs/common";
import { Observable } from "rxjs";
import { map } from "rxjs/operators";

@Injectable()
export class TransformInterceptor<T>
  implements NestInterceptor<T, Response<T>>
{
  intercept(
    _context: ExecutionContext,
    next: CallHandler
  ): Observable<Response<T>> {
    return next.handle().pipe(map((data) => ({ data })));
  }
}
```

Imagine we need to transform each occurrence of a **null** value to an empty string **''**. Implementing this with interceptors -

```typescript
// exclude-null.interceptor.ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from "@nestjs/common";
import { Observable } from "rxjs";
import { map } from "rxjs/operators";

@Injectable()
export class ExcludeNullInterceptor implements NestInterceptor {
  intercept(_context: ExecutionContext, next: Callhandler): Observable<any> {
    return next.handle().pipe(map((value) => (value === null ? "" : value)));
  }
}
```

## Exception Mapping:

Another interesting use-case is to take advantage of RxJS's `catchError()` operator to override thrown exceptions.

```typescript
// errors.interceptor.ts
import {
  BadGatewayException,
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from "@nestjs/common";
import { Observable, throwError } from "rxjs";
import { catchError } from "rxjs/operators";

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(_context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(catchError((err) => throwError(() => new BadGatewayException())));
  }
}
```

## Stream Overriding:

A simple **cache interceptor** -

```typescript
// cache.interceptor.ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from "@nestjs/common";
import { Observable, of } from "rxjs";

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  intercept(_context: ExecutionContext, next: CallHandler): Observable<any> {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
```

Our `CacheInterceptor` has a hardcoded `isCached` variable and a hardcoded response `[]` as well. The key point to note is that we return a new stream here, created by the RxJS `of()` operator, therefore the route handler **won't be called** at all. When someone calls an endpoint that makes use of `CacheInterceptor`, the response (a hardcoded, empty array) will be returned immediately. In order to create a generic solution, you can take advantage of `Reflector` and create a custom a decorator.

## More operators:

Imagine you would like to handle `timeouts` on route requests. When your endpoint doesn't return anything after a period of time, you want to terminate with an error response.

```typescript
// timeout.interceptor.ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
  RequestTimeoutException,
} from '@nestjs/common'
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(_context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError((err) => {
        if(err instanceOf TimeoutError ) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      })
    )
  }
}
```
