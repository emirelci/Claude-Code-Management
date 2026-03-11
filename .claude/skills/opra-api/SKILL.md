---
name: opra-api
description: >
  OPRA (Open Protocol for Rich APIs) ile NestJS backend API gelistirirken kullan.
  Controller olusturma, CRUD operasyonlar, filtre/siralama, entity/DTO tanimlama,
  service katmani, interceptor, transaction, context yonetimi ve module kaydi icin tetikle.
user-invocable: true
allowed-tools: Read, Grep, Glob, Edit, Write, Bash
---

# OPRA API Gelistirme Kilavuzu

Bu skill, OPRA + NestJS + SQB stack'i ile API gelistirirken uyulmasi gereken tum kalip ve konvansiyonlari icerir.

---

## 1. PROJE YAPISI

```
packages/<app-name>/src/
  modules/
    api/
      api.module.ts                       # OpraHttpModule kaydi
      controllers/
        <resource>/
          <resource>.controller.ts        # Collection (FindMany, Create)
          <resource>-single.controller.ts # Single (Get, Update, Delete)
          dto/
            create-<resource>-in.dto.ts
            update-<resource>-in.dto.ts
            index.ts
          index.ts
        healthz.controller.ts             # Health check (public)
        index.ts                          # Tum controller barrel export
    services/
      services.module.ts                  # Global service module
      services/
        <resource>.service.ts
        index.ts
  core/
    config/                               # Joi config schemalari
    constants/
```

---

## 2. ENTITY TANIMLAMA

Entity dosyalari `packages/data-models/src/entities/` altinda bulunur. Her entity hem ORM (`@sqb/connect`) hem API dokumantasyonu (`@opra/common`) dekoratorlerini birlikte kullanir.

### Temel Entity Ornegi

```typescript
import { ApiField, ComplexType, StringType } from '@opra/common';
import { Column, DataType, Entity, PrimaryKey } from '@sqb/connect';

@ComplexType({ description: 'Account' })
@Entity('accounts')
export class EntityAccount {
  @PrimaryKey()
  @Column({ fieldName: 'id', dataType: DataType.GUID, noInsert: true })
  @ApiField({ type: 'uuid', required: false, readonly: true })
  declare id: string;

  @Column({ fieldName: 'name', dataType: DataType.TEXT })
  @ApiField({ type: new StringType({ maxLength: 128 }), required: true })
  declare name: string;

  @Column({ fieldName: 'created_date', dataType: DataType.TIMESTAMP })
  @ApiField({ type: 'DateTime' })
  declare createdDate: Date;

  // Virtual field - veritabaninda yok, service tarafindan doldurulur
  @ApiField({ type: 'integer', required: false, readonly: true })
  declare tenantCount?: number;
}
```

### Kurallar

| Konu | Konvansiyon |
|------|-------------|
| Sinif adi | `Entity<PascalCase>` (ornek: `EntityAccount`, `EntityUser`) |
| Tablo adi | snake_case, cogul (`accounts`, `tenant_apps`) |
| TypeScript property | camelCase (`createdDate`, `accountId`) |
| DB column | snake_case (`created_date`, `account_id`) — `fieldName` ile belirtilir |
| Primary key | `@PrimaryKey()` + `noInsert: true` (auto-generated GUID) |
| PK disaridan ataniyorsa | `noInsert: false` (ornek: User.id Keycloak'tan gelir) |
| Property tanimi | `declare` keyword kullan (class field olarak) |

### Desteklenen DataType'lar

- `DataType.GUID` — UUID alanlar
- `DataType.TEXT` — String alanlar
- `DataType.TIMESTAMP` — Tarih/saat
- `DataType.BOOL` — Boolean
- `DataType.INTEGER` — Sayi

### Desteklenen ApiField Type'lari

- `'uuid'` — UUID format
- `'email'` — Email format
- `'DateTime'` — Tarih/saat
- `'integer'` — Tamsayi
- `'boolean'` — Boolean
- `new StringType({ maxLength: N })` — Uzunluk kisitli string
- `new BooleanType()` — Boolean tipi
- `'EnumAdi'` — Tanimlanmis enum referansi (string olarak)

### Iliskiler (Link)

```typescript
import { Link } from '@sqb/connect';

@ApiField({ readonly: true })
@(Link({ exclusive: true }).toOne(EntityAccount, {
  sourceKey: 'accountId',
  targetKey: 'id',
}))
declare account: EntityAccount;
```

- `Link.toOne()` ile one-to-one iliski tanimlanir
- `exclusive: true` kullanilir
- `sourceKey`: bu entity'deki FK alani
- `targetKey`: hedef entity'deki PK alani
- `@ApiField({ readonly: true })` ile isaret edilir

### Array Alanlar

```typescript
@Column({ fieldName: 'roles', dataType: DataType.TEXT, isArray: true })
@ApiField({ type: 'UserAccountRole', isArray: true })
declare roles: UserAccountRole[];
```

---

## 3. ENUM TANIMLAMA

```typescript
import { EnumType } from '@opra/common';

export enum InvitationStatus {
  PENDING = 'PENDING',
  ACCEPTED = 'ACCEPTED',
  REJECTED = 'REJECTED',
  EXPIRED = 'EXPIRED',
}

EnumType(InvitationStatus, {
  name: 'InvitationStatus',
  meanings: {
    PENDING: 'Pending',
    ACCEPTED: 'Accepted',
    REJECTED: 'Rejected',
    EXPIRED: 'Expired',
  },
});
```

- Enum sinifi tanimlandiktan sonra `EnumType()` fonksiyonu ile OPRA'ya kayit edilir
- `meanings` objesi her enum degerinin okunabilir aciklamasini icerir

---

## 4. DTO TANIMLAMA

DTO dosyalari controller'in `dto/` klasorunde bulunur. `@ComplexType` ve `@ApiField` dekoratorleri kullanilir.

```typescript
import { ApiField, ComplexType, StringType } from '@opra/common';

@ComplexType()
export class CreateAccountInDto {
  @ApiField({ type: new StringType({ maxLength: 128 }), required: true })
  declare name: string;
}
```

```typescript
@ComplexType()
export class CreateInvitationInDto {
  @ApiField({ type: 'uuid', required: true })
  declare accountId: string;

  @ApiField({ type: 'email', required: true })
  declare email: string;

  @ApiField({ type: new StringType({ maxLength: 32 }), required: true })
  declare firstName: string;

  @ApiField({ type: new StringType({ maxLength: 32 }), required: true })
  declare lastName: string;
}
```

### DTO Konvansiyonlari

- Input DTO isimlendirme: `Create<Resource>InDto`, `Update<Resource>InDto`
- `@ComplexType()` dekoratoru zorunlu
- `declare` keyword ile property tanimla
- DTO'lar `api.module.ts`'deki `types` dizisine eklenmelidir

---

## 5. CONTROLLER TANIMLAMA

Iki tip controller vardir: **Collection** (liste + create) ve **Single** (get + update + delete).

### 5.1 Collection Controller (FindMany + Create)

```typescript
import { HttpController, HttpOperation, OperationResult } from '@opra/common';
import { HttpContext } from '@opra/http';
import { SQBAdapter } from '@opra/sqb';

@HttpController({
  path: 'accounts',
  description: 'Accounts',
})
export class AccountController {
  constructor(private readonly accountService: AccountService) {}

  @(HttpOperation.Entity.FindMany(EntityAccount)
    .Filter('id', ['=', '!=', 'in', '!in'])
    .Filter('name', ['=', '!=', 'in', '!in'])
    .Filter('createdDate', ['=', '!=', 'in', '!in', '>', '<', '>=', '<=']))
  async findMany(ctx: HttpContext) {
    const { options } = await SQBAdapter.parseRequest(ctx);
    if (!options.count) {
      return this.service.for(ctx).findMany(options);
    }
    const { items, count } = await this.service
      .for(ctx)
      .findManyWithCount(options);
    return new OperationResult({ payload: items, totalMatches: count });
  }

  @HttpOperation.Entity.Create(EntityAccount)
  async create(ctx: HttpContext) {
    const body = await ctx.getBody<Partial<EntityAccount>>();
    return this.accountService.for(ctx).create(body as any);
  }
}
```

### 5.2 Single Controller (Get + Update + Delete)

```typescript
@(HttpController({
  path: 'account',
  description: 'Single Account',
}).KeyParam('id'))
export class AccountSingleController {
  constructor(private readonly accountService: AccountService) {}

  @HttpOperation.Entity.Get(EntityAccount)
  async getById(ctx: HttpContext) {
    const { key, options } = await SQBAdapter.parseRequest(ctx);
    return this.accountService
      .for(ctx)
      .findOne({ filter: { id: key }, ...options });
  }

  @HttpOperation.Entity.Update(EntityAccount)
  async updateById(ctx: HttpContext) {
    const { key } = await SQBAdapter.parseRequest(ctx);
    const body = await ctx.getBody<Partial<EntityAccount>>();
    return this.accountService.for(ctx).update(key, body);
  }

  @HttpOperation.Entity.Delete(EntityAccount)
  async deleteById(ctx: HttpContext) {
    const { key } = await SQBAdapter.parseRequest(ctx);
    return await this.accountService.for(ctx).delete(key);
  }
}
```

### 5.3 Custom (Non-CRUD) Operasyonlar

```typescript
@HttpController({ path: 'authz' })
export class AuthzController {
  constructor(private userService: UserService) {}

  @Public()
  @(HttpOperation.POST({
    path: 'pre-register',
  })
    .RequestContent(PreRegisterInDto)
    .UseType(PreRegisterInDto))
  async preRegister(ctx: HttpContext) {
    const body = await ctx.getBody<PreRegisterInDto>();
    // ...
    return new OperationResult({ message: 'Success', payload: result });
  }
}
```

Diger HTTP metodlari: `HttpOperation.GET({})`, `HttpOperation.POST({})`, `HttpOperation.PUT({})`, `HttpOperation.DELETE({})`

### 5.4 Health Check Controller (Public Endpoints)

```typescript
import { Public as OpraPublic } from '@opra/nestjs';
import { Public } from '@panates-prv/account-core';

@HttpController({ path: 'healthz' })
export class HealthCheckController {
  @HttpOperation.GET({ path: 'liveness' })
  @OpraPublic()
  @Public()
  async liveness() {
    return { status: 'ok', version: PACKAGE_VERSION, time: new Date().toISOString() };
  }
}
```

**Public endpointler icin iki `@Public()` dekoratoru birlikte kullanilir:**
- `@Public()` from `@panates-prv/account-core` — JWT guard'i bypass eder
- `@OpraPublic()` from `@opra/nestjs` — OPRA schema'da public olarak isaretler

### Controller Konvansiyonlari

| Konu | Konvansiyon |
|------|-------------|
| Collection path | Cogul (`accounts`, `users`, `invitations`) |
| Single path | Tekil (`account`, `user`, `invitation`) |
| KeyParam | Genelde `'id'` (UUID), bazi durumlarda `'code'` veya `'token'` |
| Constructor injection | `private readonly <service>Service: <Service>Service` |
| Her metod parametresi | `ctx: HttpContext` |
| Service cagirma | **Her zaman** `.for(ctx)` ile context bagla |
| Body okuma | `await ctx.getBody<Type>()` |
| Key (URL param) okuma | `const { key } = await SQBAdapter.parseRequest(ctx)` |
| Query params | `const { options } = await SQBAdapter.parseRequest(ctx)` |
| User bilgisi | `ctx.user?.id`, `ctx.user?.email` |
| Request params | `ctx.request.params?.paramName` |

### Filter Operatorleri

| Tip | Operatorler |
|-----|-------------|
| String/UUID | `['=', '!=', 'in', '!in']` |
| Date/Timestamp | `['=', '!=', 'in', '!in', '>', '<', '>=', '<=']` |
| Number | `['=', '!=', 'in', '!in', '>', '<', '>=', '<=']` |

### Pagination Kaliplari

`SQBAdapter.parseRequest(ctx)` otomatik olarak query parametrelerinden su alanlari cikarir:
- `limit` — sayfa basina kayit sayisi
- `offset` / `skip` — baslangic pozisyonu
- `count` — toplam kayit sayisi dahil edilsin mi (boolean)
- `projection` — dondurulecek alanlar
- `sort` — siralama
- `filter` — filtre ifadeleri

`options.count` true ise `findManyWithCount()`, degilse `findMany()` kullanilir.

---

## 6. SERVICE KATMANI

### 6.1 Temel Service

```typescript
import { Injectable } from '@nestjs/common';
import { SqbCollectionService } from '@opra/sqb';
import { SqbClient } from '@sqb/connect';

@Injectable()
export class UserService extends SqbCollectionService<EntityUser> {
  constructor(private readonly sqbClient: SqbClient) {
    super(EntityUser, { db: sqbClient });
  }
}
```

Miras alinan CRUD metodlari:
- **Read:** `findOne()`, `findById()`, `findMany()`, `findManyWithCount()`, `count()`, `exists()`
- **Create:** `create()`, `createOnly()`
- **Update:** `update()`, `updateOnly()`, `updateMany()`
- **Delete:** `delete()`, `deleteMany()`

### 6.2 Custom Is Mantigi Eklemek

```typescript
@Injectable()
export class AccountService extends SqbCollectionService<EntityAccount> {
  constructor(
    private readonly sqbClient: SqbClient,
    private readonly tenantService: TenantService,
    private readonly accountUserService: AccountUserService,
  ) {
    super(EntityAccount, { db: sqbClient });
  }

  async createAccountWithUser(accountData: Partial<EntityAccount>, userId: string) {
    const account = await this.create(accountData as any);
    await this.accountUserService.for(this.context).create({
      accountId: account.id,
      userId: userId,
      roles: [UserAccountRole.SystemAdmin],
    } as any);
    return account;
  }
}
```

**Kritik:** Baska bir service cagirilirken `.for(this.context)` ile context aktarilmalidir.

### 6.3 Interceptor (Otomatik Filtre)

Tum read/update/delete operasyonlarina otomatik filtre uygulamak icin kullanilir. Multi-tenant ve yetkilendirme icin ideal.

```typescript
interceptor(
  next: () => any,
  command: SqbEntityService.CommandInfo,
): Promise<any> {
  if (command.method === 'create') {
    return next(); // create'de filtre uygulanmaz
  }

  if (!command.options) command.options = {};

  command.options.filter = And(
    command.options.filter,
    Exists(
      Select('*')
        .from('account_users ac')
        .where(
          And(
            Eq('ac.account_id', Field('T.id')),
            Eq('ac.user_id', (this.context as HttpContext).user?.id),
          ),
        ),
    ),
  );

  return next();
}
```

- `command.method`: `'create'`, `'findOne'`, `'findMany'`, `'update'`, `'delete'` vb.
- `command.options.filter`: Mevcut filtre ile `And()` ile birlestirilir
- `T` alias'i mevcut entity tablosunu temsil eder

### 6.4 Lifecycle Hook'lari

#### `_beforeCreate` — Insert oncesi veri donusumu

```typescript
protected async _beforeCreate(
  command: SqbEntityService.CreateCommand<EntityTenantApp>,
): Promise<void> {
  const input = command.input as Partial<EntityTenantApp>;
  if (input?.createdDate) {
    const subscriptionEndDate = new Date(input.createdDate);
    subscriptionEndDate.setMonth(subscriptionEndDate.getMonth() + 12);
    input.subscriptionEndDate = subscriptionEndDate;
  }
  await super._beforeCreate(command);
}
```

#### `_create` — Tum create islemini override etme + Transaction

```typescript
protected _create(
  command: SqbEntityService.CreateCommand<EntityTenant>,
): Promise<EntityTenant> {
  return this.withTransaction(async () => {
    const result = (await super._create(command)) as EntityTenant;
    // ek is mantigi (migration, schema olusturma vb.)
    return result;
  });
}
```

Mevcut lifecycle hook'lari:
- `_beforeCreate(command)` / `_create(command)` / `_afterCreate(command, result)`
- `_beforeUpdate(command)` / `_update(command)` / `_afterUpdate(command, result)`
- `_beforeDelete(command)` / `_delete(command)` / `_afterDelete(command, result)`

### 6.5 SQB Query Builder Kullanimi

```typescript
import { op } from '@sqb/builder';

// Basit filtre (obje syntax)
await this.findOne({ filter: { id: key } });
await this.findOne({ filter: { email: data.email, status: 'ACTIVE' } });

// Karmasik filtre (builder syntax)
await this.findOne({
  filter: op.and(
    op.eq('email', data.email),
    op.eq('accountId', data.accountId),
    op.in('status', [InvitationStatus.PENDING, InvitationStatus.EXPIRED]),
  ),
});

// Interceptor icinde subquery
import { And, Eq, Exists, Field, Select } from '@sqb/builder';

Exists(
  Select('*')
    .from('account_users ac')
    .where(And(
      Eq('ac.account_id', Field('T.id')),
      Eq('ac.user_id', userId),
    ))
);
```

### 6.6 Transaction

```typescript
return this.withTransaction(async () => {
  const result = await super._create(command);
  // ek islemler...
  return result;
});
```

Hata durumunda tum islemler otomatik rollback olur.

### 6.7 Context Erisimi

```typescript
// Service icinde mevcut kullanici bilgisi
const user = (this.context as HttpContext)?.user;
const userId = (this.context as HttpContext).user?.id;

// Baska service'e context aktarimi
await this.accountUserService.for(this.context).create({...});
await this.tenantService.for(this).count({...}); // this da gecerli
```

---

## 7. MODULE KAYDI

### 7.1 Services Module

```typescript
import { Global, Module } from '@nestjs/common';
import * as services from './services/index.js';

@Module({
  imports: [EmailModule],       // Bagimli moduller
  providers: Object.values(services),
  exports: Object.values(services),
})
@Global()
export class ServicesModule {}
```

- `@Global()` ile tum uygulama genelinde erisim saglanir
- Barrel export (`index.ts`) uzerinden dinamik kayit

### 7.2 API Module

```typescript
import { OpraHttpModule } from '@opra/nestjs-http';
import { models } from '@panates-prv/account-data-models';
import * as controllers from './controllers/index.js';

@Module({
  imports: [
    PassportModule.register({ defaultStrategy: 'jwt' }),
    OpraHttpModule.forRoot({
      name: 'Account API App',
      info: { title: 'Account API App', version: PACKAGE_VERSION },
      types: [
        ...Object.values(models),    // Entity ve enum modelleri
        CreateAccountInDto,            // DTO'lar ayri eklenir
        UpdateAccountInDto,
      ],
      controllers: Object.values(controllers),
      schemaIsPublic: true,
    }),
  ],
  providers: [JwtStrategy],
})
export class ApiModule {}
```

**Onemli:** Yeni bir DTO olusturuldiginda `types` dizisine eklenmesi gerekir.

### 7.3 Barrel Export Yapisi

Her controller klasoru ve services klasoru bir `index.ts` dosyasi icerir:

```typescript
// controllers/<resource>/index.ts
export * from './<resource>.controller.js';
export * from './<resource>-single.controller.js';

// controllers/index.ts
export * from './account/index.js';
export * from './user/index.js';
export * from './healthz.controller.js';
// ...
```

Import path'lerinde `.js` uzantisi kullanilir (ESM gerekliligi).

---

## 8. RESPONSE YAPISI

### Standart CRUD Response

- **Get/Create/Update**: Entity objesi dogrudan dondurulur
- **Delete**: Silinen kaydin id'si dondurulur
- **FindMany (count yok)**: Entity dizisi dogrudan dondurulur
- **FindMany (count var)**: `OperationResult` kullanilir

```typescript
import { OperationResult } from '@opra/common';

// Paginated response
return new OperationResult({
  payload: items,
  totalMatches: count,
});

// Custom operation response
return new OperationResult({
  message: 'Success',
  payload: result,
});
```

---

## 9. YENI RESOURCE EKLEME ADIMLARI

Yeni bir resource (ornek: `Product`) eklemek icin sirasi ile:

### Adim 1: Entity tanimla
`packages/data-models/src/entities/product.entity.ts` olustur.
`@ComplexType`, `@Entity`, `@PrimaryKey`, `@Column`, `@ApiField` dekoratorlerini kullan.
`packages/data-models/src/entities/index.ts`'e export ekle.

### Adim 2: (Opsiyonel) Enum tanimla
`packages/data-models/src/enums/` altinda enum dosyasi olustur.
`EnumType()` ile kayit et. Index'e export ekle.

### Adim 3: Service olustur
`packages/<app>/src/modules/services/services/product.service.ts` olustur.
`SqbCollectionService<EntityProduct>` extend et.
`services/index.ts`'e export ekle.

### Adim 4: DTO olustur
`packages/<app>/src/modules/api/controllers/product/dto/` altinda DTO'lari olustur.
`@ComplexType` ve `@ApiField` kullan.

### Adim 5: Collection Controller olustur
`product.controller.ts` — FindMany + Create operasyonlari.
Path: cogul (`products`).

### Adim 6: Single Controller olustur
`product-single.controller.ts` — Get + Update + Delete operasyonlari.
Path: tekil (`product`), `.KeyParam('id')` ile.

### Adim 7: Controller index'e ekle
`controllers/product/index.ts` ve `controllers/index.ts` dosyalarini guncelle.

### Adim 8: API Module'e DTO kaydet
`api.module.ts`'deki `OpraHttpModule.forRoot({ types: [...] })` dizisine yeni DTO'lari ekle.

### Adim 9: DB Migration olustur
`packages/db-migrator/src/migrations/` altinda tablo olusturma migration'i ekle.

---

## 10. IMPORT REFERANSI

```typescript
// OPRA Common — Dekoratorler ve tipler
import { HttpController, HttpOperation, OperationResult, ApiField, ComplexType, StringType, EnumType } from '@opra/common';

// OPRA HTTP — Context
import { HttpContext } from '@opra/http';

// OPRA SQB — Adapter
import { SQBAdapter, SqbCollectionService, SqbEntityService } from '@opra/sqb';

// OPRA NestJS — Public dekoratoru
import { Public as OpraPublic } from '@opra/nestjs';

// SQB Connect — ORM dekoratorleri
import { Column, DataType, Entity, Link, PrimaryKey } from '@sqb/connect';
import { SqbClient } from '@sqb/connect';

// SQB Builder — Query builder
import { op } from '@sqb/builder';
import { And, Eq, Exists, Field, Select } from '@sqb/builder';

// NestJS
import { Injectable } from '@nestjs/common';
import { Global, Module } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

// Core — JWT guard
import { Public } from '@panates-prv/account-core';

// Data Models
import { EntityAccount, EntityUser, ... } from '@panates-prv/account-data-models';
import { models } from '@panates-prv/account-data-models';