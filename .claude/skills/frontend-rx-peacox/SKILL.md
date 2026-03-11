---
name: new-component
description: rx-peacox kütüphanesine yeni bir React bileşeni ekle. Bileşen dosyası, stil variants dosyası ve Storybook stories dosyası oluşturur; tüm proje konvansiyonlarına (tailwind-variants, pcx: prefix, Ant Design, Storybook) uyar.
argument-hint: <ComponentName> [açıklama]
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
disable-model-invocation: true
---

# rx-peacox: Yeni Bileşen Oluştur

`$ARGUMENTS` argümanından bileşen adını al. İlk kelime bileşen adı, geri kalanı opsiyonel açıklama.
Aşağıdaki tüm adımlarda `{ComponentName}` yerine bu bileşen adını kullan.

## 1. Önce Mevcut Yapıyı İncele

Başlamadan önce şunları oku:
- `src/components/Card/Card.tsx` — bileşen dosyası örneği
- `src/components/Card/styles/Card.styles.ts` — tv() kullanım örneği
- `src/components/FlagCard/FlagCard.variants.ts` — slots'lu tv() örneği
- `src/components/Card/Card.stories.tsx` — Storybook story örneği
- `src/components/index.ts` — mevcut export listesi

## 2. Oluşturulacak Dosyalar

### `src/components/{ComponentName}/{ComponentName}.tsx`

Ana bileşen dosyası. Şu kurallara uy:

```tsx
// Import sırası: harici kütüphaneler → antd → @ant-design/icons → local styles → local types
import { ... } from "antd";                          // Ant Design bileşenleri (gerekirse)
import { ... } from "@ant-design/icons";             // İkonlar (gerekirse)
import { type ReactNode } from "react";
import { wrapper, ... } from "./{ComponentName}.variants";  // Stil fonksiyonları

// Props interface: "I" prefix YOK, sadece Props suffix
interface {ComponentName}Props {
  className?: string;
  children?: ReactNode;
  // ... diğer proplar
}

// Named export kullan (default export YOK)
export function {ComponentName}({ className, children, ...props }: {ComponentName}Props) {
  return (
    <div className={`${wrapper()} ${className ?? ""}`}>
      {children}
    </div>
  );
}
```

**Kurallar:**
- `export default` KULLANMA — sadece named export
- Props tipi `interface` olarak tanımla, `src/components/{ComponentName}/index.ts`'ten export et
- Inline Tailwind class kullanırsan `pcx:` prefix ZORUNLU (örn: `pcx:flex pcx:gap-2`)
- Ant Design bileşenlerini doğrudan kullan (sarmalamak zorunda değilsin)
- Türkçe string sabitleri kullanabilirsin (proje dili Türkçe)

### `src/components/{ComponentName}/{ComponentName}.variants.ts`

Tüm stiller `tailwind-variants` (`tv()`) ile tanımlanır:

```ts
import { tv } from "tailwind-variants";

// Basit bileşen için (slots yok)
export const wrapper = tv({
  base: "pcx:relative pcx:w-full ...",
  variants: {
    size: {
      sm: "pcx:text-sm",
      md: "pcx:text-base",
    },
    disabled: {
      true: "pcx:opacity-50 pcx:cursor-not-allowed",
    },
  },
  defaultVariants: {
    size: "md",
  },
});

// Birden fazla element için slots kullan (FlagCard örneği gibi)
export const myComponent = tv({
  slots: {
    base: "pcx:...",
    header: "pcx:...",
    body: "pcx:...",
  },
  variants: {
    color: {
      primary: { base: "pcx:bg-blue-500" },
      secondary: { base: "pcx:bg-gray-500" },
    },
  },
});
```

**Kurallar:**
- Tüm Tailwind class'ları `pcx:` prefix ile başlar — İSTİSNASIZ
- Her ayrı element için ayrı `tv()` export et (Card gibi) VEYA slots kullan (FlagCard gibi)
- `tailwind-merge` gerekirse import et: `import { twMerge } from "tailwind-merge"`
- Renk değerleri için Tailwind token'ları tercih et, zorunluysa `[]` arbitrary kullan

### `src/components/{ComponentName}/{ComponentName}.stories.tsx`

```tsx
import type { Meta, StoryObj } from "@storybook/react-vite";
import { {ComponentName} } from "./{ComponentName}";

const meta: Meta<typeof {ComponentName}> = {
  title: "{ComponentName}/{ComponentName}",  // klasör/isim formatı
  component: {ComponentName},
  tags: ["autodocs"],
  parameters: {
    layout: "centered",
  },
  argTypes: {
    // variant proplar için control tanımla
    // örn: size: { control: "select", options: ["sm", "md", "lg"] }
  },
};

export default meta;
type Story = StoryObj<typeof {ComponentName}>;

export const Default: Story = {
  args: {
    // varsayılan prop değerleri
  },
};

// Önemli state/varyantlar için ayrı Story'ler ekle
```

**Kurallar:**
- Import: `from "@storybook/react-vite"` (NOT `@storybook/react`)
- En az bir `Default` story zorunlu
- Her önemli variant için ayrı story yaz
- State'li bileşenler için `render: (args) => { const [x, setX] = useState(...); return <.../>; }` kullan

### `src/components/{ComponentName}/index.ts`

```ts
export * from "./{ComponentName}";
```

## 3. Mevcut Export'a Ekle

`src/components/index.ts` dosyasına ekle:

```ts
export * from './{ComponentName}'
```

## 4. Kalite Kontrol

Dosyaları oluşturduktan sonra çalıştır:

```bash
npm run type-check
npm run lint
```

Hata varsa düzelt. Build çalıştırmana gerek yok.

## 5. Özet Rapor

İşlem bitince kullanıcıya şunu bildir:
- Oluşturulan dosyalar (yollarıyla birlikte)
- Props listesi
- Storybook'ta nasıl görüntüleneceği (`npm run storybook` → title path)
- Varsa dikkat edilmesi gereken özel notlar
