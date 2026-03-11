---
name: peacox
description: >
  isg-peacox kurumsal React bileşen kütüphanesi üzerinde çalışırken kullan.
  Yeni bileşen oluşturma, tema özelleştirme, Storybook hikayesi yazma,
  bileşen güncelleme veya mimari kararlar için tetikle.
---

# isg-peacox Bileşen Kütüphanesi

`@panates-prv/isg-peacox` — Panates grup şirketleri için kurumsal React bileşen kütüphanesi.
Ant Design 5 üzerine merkezi tema sistemi, Tailwind CSS (`ipcx:` öneki) ve TipTap zengin metin düzenleyici içerir.

## Temel Kurallar

### 1. Tailwind CSS — `ipcx:` Öneki Zorunlu
Tüketici uygulamalarla çakışmayı önlemek için **her** Tailwind sınıfı `ipcx:` önekiyle yazılır:
```tsx
// ✅ Doğru
<div className="ipcx:flex ipcx:items-center ipcx:gap-2 ipcx:text-sm" />

// ❌ Yanlış
<div className="flex items-center gap-2 text-sm" />
```

### 2. Ant Design — Direkt Kullanım, Tema ile Özelleştirme
Ant Design bileşenlerini doğrudan kullan; görsel özelleştirme için `src/config/themeConfig.ts` dosyasını düzenle.
Bileşen içine inline style veya CSS override ekleme — bunlar `themeConfig.ts`'e aittir.

### 3. Provider Zorunluluğu
`IsgPeacoxProvider` Ant Design `ConfigProvider`'ı sarmalar. Bu tüketici uygulamalarda zorunludur; kütüphane içinde test/storybook setup'larında kullanılır.

---

## Yeni Bileşen Oluşturma

Her bileşen kendi dizininde üç dosyayla oluşturulur:

```
src/components/BilesenAdi/
├── BilesenAdi.tsx        ← Bileşen implementasyonu
├── BilesenAdi.stories.tsx ← Storybook hikayeleri
└── index.ts              ← Re-export
```

### Bileşen Şablonu (`BilesenAdi.tsx`)
```tsx
import type { ReactNode } from "react";

export interface BilesenAdiProps {
  // props burada
}

export function BilesenAdi({ ...props }: BilesenAdiProps) {
  return (
    <div className="ipcx:...">
      {/* içerik */}
    </div>
  );
}
```

### Index Şablonu (`index.ts`)
```ts
export { BilesenAdi } from "./BilesenAdi";
export type { BilesenAdiProps } from "./BilesenAdi";
```

### Ana Export'a Ekleme (`src/components/index.ts`)
```ts
export * from "./BilesenAdi";
```

---

## Storybook Hikayesi Şablonu

```tsx
import type { Meta, StoryObj } from "@storybook/react-vite";
import { BilesenAdi } from "./BilesenAdi";

const meta: Meta<typeof BilesenAdi> = {
  title: "BilesenAdi",
  component: BilesenAdi,
  tags: ["autodocs"],
};
export default meta;

type Story = StoryObj<typeof BilesenAdi>;

/** Açıklama — JSDoc yorumu otomatik dokümana dönüşür */
export const Default: Story = {
  args: {
    // default prop değerleri
  },
};
```

Etkileşimli state gerektiren hikayeler için `render` fonksiyonu kullan (Modal ve ListWrapper hikayelerine bak).

---

## Tema Özelleştirme

Tüm tema değişiklikleri `src/config/themeConfig.ts`'te yapılır:

```ts
export const isgThemeConfig = {
  token: {
    colorPrimary: "#000000",   // Marka rengi
    // ... diğer global token'lar
  },
  components: {
    Button: { /* Ant Design Button override */ },
    Input:  { /* Ant Design Input override */ },
    // ...
  },
};
```

Yeni bir Ant Design bileşeni için component-level override eklemek: `components` nesnesine bileşen adını key olarak ekle.

---

## Mevcut Bileşenler

| Bileşen | Açıklama |
|---------|----------|
| `PageHeader` | Sayfa başlığı — title, description, actions, profile desteği |
| `Modal` | Form modal — Ant Design Form entegrasyonlu, validasyon aware |
| `ListWrapper` | Sayfalı liste sarmalayıcı — render prop ile generic tip |
| `TextArea` | TipTap tabanlı zengin metin düzenleyici |
| `AiChatWidget` | Yapay zeka sohbet widget'ı |
| `AntEditor` | Ant Design tema editörü |
| `DropDownItems/PersonItem` | Açılır menü kişi satırı |

---

## Build ve Geliştirme Komutları

```bash
npm run storybook      # Bileşen geliştirme ortamı (port 6006)
npm run build          # Kütüphane build (output: build/)
npm run qc             # Kalite kontrolü (lint + type-check)
npm run lint:fix       # Otomatik lint düzeltme
npm run type-check     # TypeScript kontrolü
npm run circular-check # Dairesel bağımlılık kontrolü
```

## Sürüm Yayınlama

```bash
npm version patch      # Versiyon artır
git push origin main --tags  # GitHub Actions tetikle
```

---

## TypeScript Kuralları

- `strict: true` — tüm strict kontroller aktif
- Kullanılmayan `import` ve değişkenler yasak (`noUnusedLocals`, `noUnusedParameters`)
- Bileşen props'ları için daima `interface` tanımla ve dışa aktar
- Ant Design bileşenlerini extend ederken `Omit<AntProps, "overridden-prop">` kullan (Modal'a bak)
- Generic bileşenler için `<T>` kullan (ListWrapper ve Modal örneklerine bak)
