# Providers

## Services:

### `cats.service.ts`:

```typescript
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface.ts';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  create(cat: Cat) {
    this.cats.push(cat);
  }

  findAll(): Cat[] {
    return this.cats;
  }
}
```

### `cats.interface.ts`:

```typescript
export interface Cat {
  name: string;
  age: number;
  breed: string;
}
```

### `cats.controller.ts`:

```typescript
import { Controller, Get, Post } from '@nestjs/common';
import { CreateCatDto } from './dtos/create-cat.dto.ts';
import { CatsService } from './cats.service.ts';
import { Cat } from './interfaces/cat.interface.ts';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Post()
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
```

## Dependancy Injection:

```typescript
constructor(private catsService: CatsService) {}
```

## Optional Providers:

```typescript
import { Injectable, Optional, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  constructor(@Optional() @Inject('HTTP_OPTIONS') private httpClient: T) {}
}
```

## Property-based Injection:

```typescript
import { Injectable, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  @Inject('HTTP_OPTIONS')
  private readonly httpClient: T;
}
```

## Provider Registration:

```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```
