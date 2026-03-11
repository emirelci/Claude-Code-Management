---
name: opra-api
description: >
  OPRA (Open Protocol for Rich APIs) ile backend API gelistirirken kullan.
  Controller olusturma, operasyon tanimlama, filtre/siralama, dosya yukleme,
  nested controller, parametre yonetimi ve response yapilandirmasi icin tetikle.
user-invocable: true
allowed-tools: Read, Grep, Glob, Edit, Write, Bash
---

# OPRA Backend API Development Guide

Bu skill, OPRA framework kullanarak NestJS tabanlı REST API'ler geliştirmek için referans sağlar.

## Paketler ve Import'lar

```typescript
// Temel dekoratörler ve tipler
import {
  HttpController, HttpOperation, ApiField, ComplexType,
  EnumType, StringType, PickType, PartialType, OmitType,
  ArrayType, ForbiddenError, BadRequestError, OperationResult,
  OpraFilter, HttpStatusCode, MimeTypes, HttpParameter,
} from '@opra/common';

// HTTP context
import { HttpContext, MultipartReader } from '@opra/http';

// Veritabanı adaptörleri
import { SQBAdapter } from '@opra/sqb';       // SQL veritabanları
import { MongoAdapter } from '@opra/mongodb';  // MongoDB

// NestJS entegrasyonu
import { OpraHttpModule } from '@opra/nestjs-http';
import { Public } from '@opra/nestjs';

// SQB ORM (SQL entity'leri için)
import { Column, DataType, Entity, Link, PrimaryKey } from '@sqb/connect';
import { op } from '@sqb/builder';
```

## Controller Türleri

OPRA'da üç temel controller kalıbı vardır:

### Collection Controller — Kayıt listesi yöneten

```typescript
@HttpController({
  path: 'Customers',         // URL: /Customers
  description: 'Müşteriler',
})
export class CustomersController {

  @HttpOperation.Entity.Create(Customer, {
    requestBody: { type: OmitType(Customer, ['_id']) },
  })
  async create(context: HttpContext) { ... }

  @(HttpOperation.Entity.FindMany(Customer)
    .Filter('name', ['=', '!=', 'like', '!like'])
    .SortFields('_id', 'name', 'createdAt')
    .DefaultSort('name'))
  async findMany(context: HttpContext) { ... }

  @HttpOperation.Entity.UpdateMany(Customer)
  async updateMany(context: HttpContext) { ... }

  @HttpOperation.Entity.DeleteMany(Customer)
  async deleteMany(context: HttpContext) { ... }
}
```

### Singleton/Entity Controller — Tekil kaynak yöneten

`KeyParam` URL'e `@:paramAdi` ekler: `/Customers@:customerId`

```typescript
@(HttpController({
  path: 'Customers',
}).KeyParam('customerId', 'number'))   // → /Customers@:customerId
export class CustomerController {

  @HttpOperation.Entity.Get(Customer)
  async get(context: HttpContext) { ... }

  @HttpOperation.Entity.Update(Customer)
  async update(context: HttpContext) { ... }

  @HttpOperation.Entity.Delete(Customer)
  async delete(context: HttpContext) { ... }
}
```

### Custom Controller — Serbest endpoint'ler

```typescript
@HttpController({ path: 'auth', description: 'Kimlik doğrulama' })
export class AuthController {

  @(HttpOperation({ path: 'login' })
    .QueryParam('user', String)
    .QueryParam('password', 'string')
    .Response(200, { type: OperationResult }))
  async login(context: HttpContext) {
    return new OperationResult({ message: 'Giriş başarılı' });
  }

  @(HttpOperation({ path: '/logout' })
    .Response(200, { type: OperationResult }))
  async logout() {
    return new OperationResult({ message: 'Çıkış yapıldı' });
  }
}
```

## @HttpController Seçenekleri

```typescript
@HttpController({
  path: 'Customers',          // URL path segmenti (zorunlu)
  name: 'Customers',          // İsim (yoksa sınıf adından otomatik türetilir)
  description: 'Açıklama',   // API dokümantasyonu için
  controllers: [              // Nested (alt) controller'lar
    (parent: ParentCtrl) => new ChildController(parent.db),
  ],
})
```

**Zincirlenebilir metotlar** — class seviyesinde tüm operasyonlara uygulanır:

| Metot | Açıklama |
|-------|----------|
| `.Header('name', type)` | Header parametresi tanımla |
| `.QueryParam('name', type)` | Query string parametresi |
| `.PathParam('name', type)` | Path parametresi (`/path/:name`) |
| `.KeyParam('name', type)` | Birincil anahtar parametresi (`/path@:name`) |
| `.Cookie('name', type)` | Cookie parametresi |
| `.UseType(SomeClass)` | Bu controller'da kullanılacak ek tipleri kaydet |

> **İsim kuralı:** `CustomersController` → isim `Customers` olarak otomatik alınır.

### Nested Controller

Alt controller'lar `controllers` array'i ile tanımlanır.
Parent'ın `KeyParam`'ı alt controller URL'inde de geçerlidir.

```typescript
// Parent controller: /Customers@:customerId
@(HttpController({
  path: 'Customers',
  controllers: [
    (parent: CustomerController) => new CustomerNotesController(parent.db),
  ],
}).KeyParam('customerId', 'number'))
export class CustomerController { ... }

// Child controller: /Customers@:customerId/Notes
@HttpController({ path: 'Notes' })
export class CustomerNotesController {

  // KeyParam operasyon seviyesinde de tanımlanabilir:
  @(HttpOperation.Entity.Get(Note).KeyParam('_id', Number))
  async get(context: HttpContext) {
    // Parent path param'ına erişim:
    const customerId = context.pathParams.customerId;
    const noteId = context.pathParams._id;
    ...
  }

  @(HttpOperation.Entity.FindMany(Note)
    .SortFields('_id', 'title')
    .DefaultSort('_id')
    .Filter('_id')
    .Filter('title'))
  async findMany(context: HttpContext) {
    const customerId = context.pathParams.customerId;
    ...
  }
}
```

## Entity Operasyonları

`HttpOperation.Entity.*` — REST kaynak operasyonları için hazır decorator'lar:

| Decorator | HTTP Metodu | Açıklama |
|-----------|-------------|----------|
| `Entity.Create(Type, opts)` | POST | Yeni kayıt oluştur |
| `Entity.Get(Type)` | GET | Tek kayıt getir |
| `Entity.FindMany(Type)` | GET | Liste getir |
| `Entity.Update(Type)` | PATCH | Kısmi güncelle |
| `Entity.Replace(Type)` | PUT | Tam güncelle |
| `Entity.Delete(Type)` | DELETE | Tek kayıt sil |
| `Entity.UpdateMany(Type)` | PATCH | Toplu güncelle |
| `Entity.DeleteMany(Type)` | DELETE | Toplu sil |

### Entity CRUD Örnekleri

```typescript
// CREATE - POST
@HttpOperation.Entity.Create(User)
async create(ctx: HttpContext) {
  const { data, options } = await SQBAdapter.parseRequest(ctx);
  return this.userService.for(ctx).create(data, options);
}

// GET - tek kayıt (KeyParam gerektirir)
@HttpOperation.Entity.Get(User)
async get(ctx: HttpContext) {
  const { key, options } = await SQBAdapter.parseRequest(ctx);
  return this.userService.for(ctx).findById(key, options);
}

// FIND MANY - liste
@HttpOperation.Entity.FindMany(User)
async findMany(ctx: HttpContext) {
  const { options } = await SQBAdapter.parseRequest(ctx);
  return this.userService.for(ctx).findMany(options);
}

// UPDATE - PATCH
@HttpOperation.Entity.Update(User)
async update(ctx: HttpContext) {
  const { key, data, options } = await SQBAdapter.parseRequest(ctx);
  return this.userService.for(ctx).update(key, data, options);
}

// DELETE
@HttpOperation.Entity.Delete(User)
async delete(ctx: HttpContext) {
  const { key } = await SQBAdapter.parseRequest(ctx);
  return this.userService.for(ctx).delete(key);
}
```

### Request Body Kısıtlama

```typescript
@HttpOperation.Entity.Create(User, {
  requestBody: {
    type: PickType(User, ['name', 'email', 'password']),
  },
})
```

### Custom Operasyonlar

```typescript
@HttpOperation.POST({ path: '/sign-in' })
.RequestContent(SignInRequestDto)
async signIn(ctx: HttpContext) {
  const data = await ctx.getBody<SignInRequestDto>();
  return this.authService.signIn(data.email, data.password);
}

@HttpOperation.GET({ path: '/leave-usage' })
.QueryParam('year', { type: 'number' })
async getReport(ctx: HttpContext): Promise<ReportResponse> {
  const { year } = ctx.queryParams as { year?: string };
  return this.reportService.for(ctx).getReport({ year: parseInt(year, 10) });
}

@(HttpOperation({ path: '/Approve', description: 'Approves the enrollment' })
  .QueryParam('notes')
  .Response(HttpStatusCode.OK, { type: PractitionerEnrollmentEvent }))
async approve(context: HttpContext) {
  const enrollmentId = context.pathParams.enrollmentId;
  return service.approve(enrollmentId, { notes: context.queryParams.notes });
}
```

## Filtre ve Sıralama

FindMany operasyonlarına zincirlenmiş dekoratörler olarak eklenir:

```typescript
@(HttpOperation.Entity.FindMany(PractitionerEnrollment)
  .Filter('_id', ['=', '!=', 'in', '!in'])
  .Filter('status', ['=', '!=', 'in', '!in'])
  .Filter('enrollDate', ['=', '!=', '>', '<', '>=', '<='])
  .Filter('applicant._display', ['=', '!=', 'like', '!like', '>', '<', '>=', '<='])
  .SortFields('status', 'enrollDate', 'applicant._display', 'birthDate'))
async findMany(context: HttpContext) {
  const { options } = await MongoAdapter.parseRequest(context);
  return this.service.for(context).findMany(options);
}
```

**Desteklenen filtre operatörleri:**
- Eşitlik: `=`, `!=`
- Karşılaştırma: `<`, `>`, `>=`, `<=`
- Liste: `in`, `!in`
- Metin: `like`, `!like`, `ilike` (case-insensitive), `!ilike`

> **Not:** `.Filter('status')` — operatör belirtilmezse sadece `=` kabul edilir.
> `.DefaultSort('name')` — varsayılan sıralama belirtmek için kullanılır.

### Programatik Filtre (SQB)

```typescript
import { op } from '@sqb/builder';
const combinedFilter = op.and(options.filter, op.eq('departmentId', departmentId));
```

### Programatik Filtre (MongoDB)

```typescript
import { OpraFilter } from '@opra/common';
const filter = MongoAdapter.prepareFilter([
  options.filter,
  OpraFilter.$eq('organization._id', organizationId),
]);
```

## Parametre Yönetimi

Hem `@HttpController` hem de `@HttpOperation` üzerinde aynı API ile çalışır:

```typescript
// Operasyon seviyesinde
@(HttpOperation({ path: 'search' })
  .QueryParam('q', String)                           // Tip: String constructor
  .QueryParam('limit', 'number')                     // Tip: string olarak
  .QueryParam('active', { type: Boolean, required: true })  // Options objesi
  .PathParam('id', 'number')
  .Header('Authorization', String)
  .Cookie('sessionId', 'string'))
async search(context: HttpContext) {
  const { q, limit } = context.queryParams;
  const { id } = context.pathParams;
  const auth = context.headers['Authorization'];
}
```

### QueryParam Örnekleri

```typescript
@(HttpOperation.Entity.FindMany(LeaveRequest)
  .QueryParam('myLeaves', Boolean)
  .QueryParam('ownDepartmentLeaves', Boolean))
async gets(ctx: HttpContext) {
  const { options } = await SQBAdapter.parseRequest(ctx);
  options.params = {
    ...(options.params ?? {}),
    myLeaves: ctx.queryParams.myLeaves,
    ownDepartmentLeaves: ctx.queryParams.ownDepartmentLeaves,
  };
  return this.service.for(ctx).findMany(options);
}
```

### Custom Parametre Tipleri

```typescript
export function LanguageType(required: boolean = true): HttpParameter.Options {
  return {
    required,
    type: new StringType({
      pattern: `^(\\*|[a-z]{2,3}(-[A-Z]{2})?)$`,
      patternName: 'Language-Code',
    }),
  };
}

// Kullanım:
@(HttpController({ path: 'auth' }).QueryParam('language', LanguageType()))
```

## Entity Tanımlama

### SQL Entity (SQB + OPRA)

```typescript
@Entity('users')
@ComplexType({ description: 'User entity' })
export class User {
  @ApiField({ readonly: true })
  @PrimaryKey()
  @Column({ dataType: DataType.GUID, fieldName: 'user_id', noInsert: true, noUpdate: true })
  declare userId: string;

  @ApiField({ required: true })
  @Column({ dataType: DataType.TEXT, fieldName: 'email' })
  declare email: string;

  @ApiField({ required: true, type: new StringType({ maxLength: 255 }) })
  @Column({ dataType: DataType.TEXT, fieldName: 'full_name' })
  declare fullName: string;

  @ApiField()
  @Column({ dataType: DataType.TEXT, fieldName: 'phone' })
  declare phone?: string;

  @ApiField({ required: true, isArray: true })
  @Column({ dataType: DataType.TEXT, fieldName: 'roles', enum: Role, isArray: true })
  declare roles: Role[];

  @ApiField({ readonly: true })
  @(Link({ exclusive: true }).toOne(() => Department, {
    sourceKey: 'departmentId',
    targetKey: 'departmentId',
  }))
  declare department: Department;
}
```

### ApiField Seçenekleri

- `required: boolean` - zorunlu alan
- `readonly: boolean` - güncelenemez
- `type: TypeDefinition` - özel tip (StringType, NumberType, vb.)
- `description: string` - açıklama
- `default: any` - varsayılan değer
- `isArray: boolean` - dizi

### Column DataType Değerleri

`DataType.GUID`, `DataType.TEXT`, `DataType.VARCHAR`, `DataType.NUMBER`, `DataType.DATE`, `DataType.TIMESTAMP`, `DataType.BOOL`

### Enum Tanımlama

```typescript
import { EnumType } from '@opra/common';

export enum Role {
  ADMIN = 'Admin',
  MANAGER = 'Manager',
  EMPLOYEE = 'Employee',
}

EnumType(Role, {
  name: 'role',
  meanings: {
    ADMIN: 'Admin User',
    MANAGER: 'Manager Role',
    EMPLOYEE: 'Employee Role',
  },
});
```

## Dosya Yükleme (Multipart)

```typescript
import { HttpOperation } from '@opra/common';
import { HttpContext, MultipartReader } from '@opra/http';

@(HttpController({ path: 'avatar' }).UseType(AvatarMetadata))
export class AvatarController {

  @(HttpOperation.POST({})
    .MultipartContent({}, content => {
      content.Field('name',     { type: String, required: true });
      content.Field('metadata', { type: AvatarMetadata });
      content.File('image',     { contentType: 'image/*', required: true });
    })
    .Response(200, { type: AvatarMetadata }))
  async upload(context: HttpContext) {
    const reader = await context.getMultipartReader();
    const parts  = await reader.getAll();

    const imagePart = parts.find(p => p.field === 'image') as MultipartReader.FileInfo;
    const namePart  = parts.find(p => p.field === 'name')  as MultipartReader.FieldInfo;
    // imagePart.stream — dosya stream'i
    // namePart.value   — alan değeri
  }
}
```

### Multipart ile Limit Kullanımı

```typescript
@(HttpOperation.POST()
  .MultipartContent(
    { maxFiles: 1, maxFields: 0, maxTotalFileSize: 1024 * 1024 * 10 },
    subContent => {
      subContent.File('file', { required: true });
      subContent.Field('info', {
        type: PartialType(PickType(Attachment, ['title', 'description', 'contentLanguage'])),
        required: true,
      });
    },
  )
  .Response(HttpStatusCode.OK, { type: Attachment }))
async post(context: HttpContext) {
  const reader = await context.getMultipartReader();
  const parts = await reader.getAll();
  const filePart = parts.find(x => x.kind === 'file' && x.field === 'file') as MultipartReader.FileInfo;
  const infoPart = parts.find(x => x.kind === 'field' && x.field === 'info') as MultipartReader.FieldInfo;
  return this.service.addAttachment(filePart.storedPath, {
    contentType: filePart.mimeType,
    ...infoPart.value,
  });
}
```

## Response Yapılandırması

```typescript
@(HttpOperation({ path: 'report' })
  .Response(200, {
    type: ReportResult,
    description: 'Başarılı',
    contentType: 'application/json',
  })
  .Response(422, {
    description: 'Doğrulama hatası',
    contentType: 'application/opra+json',
  }))
async getReport(context: HttpContext) {
  return new OperationResult({
    payload: items,
    totalMatches: count,
    message: 'OK',
  });
}
```

## OperationResult

Liste sonuçları veya mesaj döndürmek için kullanılır:

```typescript
import { OperationResult } from '@opra/common';

// Liste + toplam sayı
return new OperationResult({
  payload: items,
  totalMatches: count,
});

// Sadece mesaj
return new OperationResult({ message: 'İşlem başarılı' });
```

## API Module Kayıt

```typescript
@Module({
  imports: [
    OpraHttpModule.forRoot({
      imports: [ApiLoggerModule, ServicesModule],
      providers: [SomeUtils],
      exports: [SomeUtils],
      name: 'MyApi',
      info: { title: 'My API', version: '1.0' },
      references: {
        models: () => ModelsDocument.create(),
      },
      types: [...Object.values(models)],
      controllers: [
        AuthSingleton,
        UserCollection,
        UserSingleton,
      ],
      schemaIsPublic: true,
    }),
  ],
})
export class ApiModule {}
```

## HttpContext — İstek Verilerine Erişim

Her operasyon metodunun tek parametresi `context: HttpContext`'tir:

```typescript
async myMethod(context: HttpContext) {
  // Parametreler
  context.pathParams.customerId   // Path parametresi
  context.queryParams.limit       // Query parametresi
  context.headers['x-token']      // Header
  context.cookies.session         // Cookie

  // Body
  const body = await context.getBody<MyType>();   // JSON body

  // Multipart (dosya yükleme)
  const reader = await context.getMultipartReader();
  const parts = await reader.getAll();

  // Ham request/response
  context.request    // HttpIncoming
  context.response   // HttpOutgoing

  // Auth user (guard'lardan)
  context.user

  // Hata biriktirme
  context.errors.push(new Error('...'));

  // Cookie set etme
  context.response.cookie('token', value, {
    httpOnly: true, secure: true, sameSite: 'lax',
  });
}
```

## PickType, PartialType ve OmitType

```typescript
// Belirli alanları seç
PickType(User, ['name', 'email', 'password'])

// Tüm alanları opsiyonel yap
PartialType(User)

// İkisini birleştir
PartialType(PickType(User, ['name', 'gender', 'birthDate']))
```

### OmitType — Belirli Alanları Çıkarma

Create operasyonlarında `_id` gibi server-taraflı alanları body'den çıkarmak için:

```typescript
import { OmitType } from '@opra/common';

@HttpOperation.Entity.Create(Customer, {
  requestBody: {
    type: OmitType(Customer, ['_id', 'createdAt', 'updatedAt']),
  },
})
async create(context: HttpContext) { ... }
```

## Veri Tipleri Tanımlama

```typescript
import { ComplexType, ApiField, ArrayType } from '@opra/common';

@ComplexType()
export class Customer {
  @ApiField({ required: true })
  declare _id: number;

  @ApiField({ required: true })
  declare name: string;

  @ApiField({ type: ArrayType(String) })
  tags?: string[];

  @ApiField({ type: 'date' })
  birthDate?: Date;
}
```

## Ham HTTP Metot Decorator'ları

`Entity.*` kullanmak yerine ham metot belirtmek gerektiğinde:

```typescript
@HttpOperation.GET({ path: 'ping' })
async ping() { return { ok: true }; }

@HttpOperation.POST({ path: 'process' })
async process(context: HttpContext) { ... }

@HttpOperation.PUT({})
async replace(context: HttpContext) { ... }

@HttpOperation.DELETE({ path: 'purge' })
async purge(context: HttpContext) { ... }
```

---

## Hızlı Başvuru — Ne Zaman Ne Kullanılır?

| Senaryo | Kullan |
|---------|--------|
| Koleksiyon (liste, oluştur, toplu güncelle) | `CustomersController` kalıbı + `Entity.FindMany/Create/UpdateMany/DeleteMany` |
| Tekil kaynak (getir, güncelle, sil) | `CustomerController` kalıbı + `.KeyParam()` + `Entity.Get/Update/Delete` |
| Sabit tekil kaynak (profil gibi) | `MyProfileController` kalıbı (KeyParam yok) + `Entity.Get/Update/Delete` |
| Serbest endpoint (login, export vb.) | `HttpOperation({ path })` + `.QueryParam/.Response` zinciri |
| Dosya yükleme | `HttpOperation.POST().MultipartContent()` + `context.getMultipartReader()` |
| Alt kaynak (müşterinin notları) | Nested controller + `controllers` array + `context.pathParams.parentId` |
