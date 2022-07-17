# Guards:

A guard is a class annotated with the `@Injectable()` decorator, which implements the `CanActivate` interface.
Guards have a **single responsibility**. They determine wheter a given request will be handled by the route handler or not, depending on certain conditions (like permissions, roles, ACLs, etc.) present at run-time. This is often referred to as **authorization**.

> Guards are executed **after** all middlewares, but **before** any interceptor or pipe.

## Authorization Guard:

```typescript
// auth.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```

## Role-based Authentication:

```typescript
// roles.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';

@Injectable()
export class RolesGurad implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    return true;
  }
}
```

## Binding Guards:

Like pipes and exception filters, guards can be controller-scoped, method-scoped, or global-scoped.
`@UseGuards()` decorator may take a single argument, or a comma-separted list of arguments.
The `@UseGuards()` decorator is imported from `@nestjs/common` package.

```typescript
@Controller('cats')
@UseGuards(RolesGuard)
export class CatsController {}
```

```typescript
// global-scoped guard
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { RolesGuard } from './guards/roles.guard';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalGuards(new RolesGuard());
  await app.listen(3000);
}
bootstrap();
```

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';
import { RolesGuard } from './guards/roles.guard';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
    },
  ],
})
export class AppModule {}
```

## Setting Roles per Handler:

Nest provides the ability to attach custom **metadata** to route handlers through `@SetMetadata()` decorator.

```typescript
@Post()
@SetMetadata('roles', ['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

> `@SetMetadata()` decorator is imported froom `@nestjs/common` package.

With the construction above, we attached the **roles** metadata (**roles** is a key, while **['admin']** is a particular value) to the **create()** method. While this works, it's not good practice to use `@SetMetadata()` directly in routes. Instead, create custom decorators.

```typescript
// roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

expor const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

```typescript
// cats.controller.ts
@Post()
@Roles('admin')
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

## Putting it all together:

```typescript
// roles.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get<string>('roles', context.getHandler());
    if (!roles) {
      return true;
    }

    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }

  private matchRoles(roles: string[], userRoles: string[]): boolean {}
}
```

When a guard returns `false`, the framework throws a `ForbiddenException`. To return a different error response, throw specific exception -

```typescript
throw new UnauthorizedException();
```

Any exception thrown by a guard will be handled by the `exception layer`(global exception filter and any exceptions filters that are applied to the current context).
