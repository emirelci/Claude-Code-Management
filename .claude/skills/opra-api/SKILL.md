---
name: opra-api
description: >
  OPRA (Open Protocol for Rich APIs) ile backend API geliştirirken kullan.
  Controller oluşturma, operasyon tanımlama, filtre/sıralama, dosya yükleme,
  nested controller, parametre yönetimi ve response yapılandırması için tetikle.
---

# OPRA API Geliştirme Kılavuzu

OPRA, TypeScript decorator'ları ile HTTP API tanımlamayı standartlaştıran bir framework'tür.
Temel paketler: `@opra/common` (decorator'lar, tipler), `@opra/http` (HttpContext, adapter).

---

## 1. Controller Türleri

OPRA'da üç temel controller kalıbı vardır:

### Collection Controller — Kayıt listesi yöneten

```ts
import { HttpController, HttpOperation, OmitType, OperationResult } from '@opra/common';
import { HttpContext } from '@opra/http';

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

```ts
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

```ts
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

---

## 2. @HttpController Seçenekleri

```ts
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

---

## 3. Entity Operasyonları

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

---

## 4. Filtre ve Sıralama

`FindMany`, `UpdateMany`, `DeleteMany` üzerinde kullanılır:

```ts
@(HttpOperation.Entity.FindMany(Customer)
  // Alan adı + izin verilen operatörler
  .Filter('_id',      ['=', '!=', '<', '>', '>=', '<=', 'in', '!in'])
  .Filter('name',     ['=', '!=', 'like', '!like', 'ilike', '!ilike'])
  .Filter('status')   // Operatör belirtilmezse sadece '=' kabul edilir
  .Filter('active')
  // Sıralanabilir alanlar
  .SortFields('_id', 'name', 'createdAt')
  // Varsayılan sıralama
  .DefaultSort('name'))
async findMany(context: HttpContext) { ... }
```

**Desteklenen filtre operatörleri:**
- Eşitlik: `=`, `!=`
- Karşılaştırma: `<`, `>`, `>=`, `<=`
- Liste: `in`, `!in`
- Metin: `like`, `!like`, `ilike` (case-insensitive), `!ilike`

---

## 5. Parametre Tanımlama

Hem `@HttpController` hem de `@HttpOperation` üzerinde aynı API ile çalışır:

```ts
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

---

## 6. HttpContext — İstek Verilerine Erişim

Her operasyon metodunun tek parametresi `context: HttpContext`'tir:

```ts
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

  // Hata biriktirme
  context.errors.push(new Error('...'));
}
```

---

## 7. Nested (İç İçe) Controller

Alt controller'lar `controllers` array'i ile tanımlanır.
Parent'ın `KeyParam`'ı alt controller URL'inde de geçerlidir.

```ts
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

---

## 8. Dosya Yükleme (Multipart)

```ts
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

---

## 9. Response Yapılandırma

```ts
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

---

## 10. OperationResult

Liste sonuçları veya mesaj döndürmek için kullanılır:

```ts
import { OperationResult } from '@opra/common';

// Liste + toplam sayı
return new OperationResult({
  payload: items,
  totalMatches: count,
});

// Sadece mesaj
return new OperationResult({ message: 'İşlem başarılı' });
```

---

## 11. OmitType — Kısmi Tip Oluşturma

Create operasyonlarında `_id` gibi server-taraflı alanları body'den çıkarmak için:

```ts
import { OmitType } from '@opra/common';

@HttpOperation.Entity.Create(Customer, {
  requestBody: {
    type: OmitType(Customer, ['_id', 'createdAt', 'updatedAt']),
  },
})
async create(context: HttpContext) { ... }
```

---

## 12. Veri Tipleri Tanımlama

```ts
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

---

## 13. Ham HTTP Metot Decorator'ları

`Entity.*` kullanmak yerine ham metot belirtmek gerektiğinde:

```ts
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
