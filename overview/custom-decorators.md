# Custom Decorators:

| Decorator                    | Express(or Fastify) object               |
| ---------------------------- | ---------------------------------------- |
| **@Request()**, **@Req()**   | **req**                                  |
| **@Response()**, **@Res()**  | **res**                                  |
| **@Next()**                  | **next**                                 |
| **@Session()**               | **req.session**                          |
| **@Param(param?: string)**   | **req.params** / **req.params[param]**   |
| **@Body(param?: string)**    | **req.body** / **req.body[param]**       |
| **@Query(param?: string)**   | **req.query** / **req.query[param]**     |
| **@Headers(param?: string)** | **req.headers** / **req.headers[param]** |
| **@Ip()**                    | **req.ip**                               |
| **@HostParam()**             | **req.hosts**                            |

In the node.js world, it's common practice to attach properties to the **request** object.

```typescript
const user = req.user;
```

In order to make your code more readable and transparent, you can create a `@User()` decorator and reuse it across all of your controllers.

```typescript
// user.decorator.ts
import { createParamDecorator, ExecutionContext } from "@nestjs/common";

export const User = createParamDecorator(
  (data: unknown, context: ExecutionContext) => {
    const request = context.switchToHttp().getRequest();
    return request.user;
  }
);
```

```typescript
@Get()
async currentUser(@User() user: UserEntity) {
  return user;
}
```

The user entity of an authenticated user might look like -

```json
{
  "id": "101",
  "firstName": "Alan",
  "lastName": "Turing",
  "email": "alan@email.com",
  "roles": ["admin"]
}
```

Let's define a decorator that takes a property name as key, and returns the associated value if it exists (or undefined if it doesn't exist, or if the `user` object has not been created).

```typescript
// user.decorator.ts
import { createParamDecorator, ExecutionContext } from "@nestjs/common";

export const User = createParamDecorator(
  (data: string, context: ExecutionContext) => {
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return data ? user?.[data] : user;
  }
);
```

```typescript
@Get()
async findOne(@User('firstName') firstName: string) {
  console.log(`Hello ${firstName}`);
}
```

## Working with Pipes:

Nestjs treats custom param decorators in the same fashion as the built-in ones(**@Body()**, **@Param()** and **@Query()**). This means that pipes are executed for the custom annotated parameters as well.

```typescript
@Get()
async findOne(
  @User(new ValidationPipe({ validateCustomDecorators: true }))
  user: UserEntity
) {
  console.log(user);
}
```

> Note that `validateCustomDecorators` option must be set to true. `ValidationPipe` does not validate arguments annotated with the custom decorators by default.

## Decorator Composition:

Nest provides a helper method to compose multiple decorators. For example, suppose you want to combine all decorators related to authentication into a single decorator.

```typescript
// auth.decorator.ts
import { applyDecorators } from "@nestjs/common";

export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata("roles", roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: "Unauthorized" })
  );
}
```

```typescript
@Get('users')
@Auth('admin')
findAllUsers() {}
```
