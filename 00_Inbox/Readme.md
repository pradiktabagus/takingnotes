# Architecture Decisions — Refactor Arsitektur Monorepo

Format tiap keputusan: **Konteks** → **Keputusan** → **Alasan** →
**Alternatif yang dipertimbangkan**.

---

## AD-1: Prinsip layering & ownership — "Apps tipis, libs punya semua yang reusable"

**Konteks**: Layer secara konsep sudah benar (apps → view-model → service →
adapter; UI di base-ui). Tapi dalam praktik `apps/*` menyimpan orkestrasi
halaman (fetch, rakit `titlePage`/`breadcrumbs`/`currentPageData`, panggil
transform) yang dicopy-paste antar app dan sudah drift.

**Keputusan**: Tetapkan kontrak ownership eksplisit. `apps/*` hanya boleh berisi
hal yang **memang milik app itu**: struktur routing (folder Next), binding
env/config, middleware, composition root (registry provider). Semua yang
reusable — fetch, transform, rakit props, business logic, komponen — wajib
tinggal di `libs`.

| Layer | Boleh berisi | TIDAK boleh |
|---|---|---|
| `apps/*` | routing, binding env/config, middleware, composition root | fetch, transform, rakit props, business logic |
| `libs/view-model` | page loader/controller (fetch+transform+rakit opsi), hooks client | JSX komponen presentational, akses `document` di jalur server |
| `libs/base-ui` | page component presentational + `SectionRenderer` | fetch data, baca `process.env` |
| `libs/service-lib` | akses data (adapter/api/server-action) | tahu React/UI |
| `libs/global-lib` | utils, hooks, provider, server-utils | import `base-ui`/`view-model` |
| `libs/config-ui` | theme + config per-publisher/per-app | business logic |
| `libs/contracts` (BARU) | tipe **kontrak bersama**: domain data (ad config, content meta, article/poll) **+ kontrak view** (`CustomINode`, `Props` komponen) | runtime/logic, import lib lain, tipe envelope API mentah |

**Arah dependensi (satu arah, tidak boleh ke atas):**
`apps` → (`view-model`, `base-ui`, `config-ui`, `global-lib`, `contracts`);
`view-model` → (`service-lib`, `config-ui`, `global-lib`, `contracts`) — **tanpa** `base-ui`;
`base-ui` → (`view-model`, `config-ui`, `global-lib`, `contracts`);
`service-lib` → (`global-lib`, `contracts`);
`contracts` → (tidak import apa pun — pure types).

> Di bawah AD-9 **Filosofi B**: `base-ui → view-model` **dipertahankan** (komponen
> boleh mengonsumsi hook view-model), sedangkan `view-model → base-ui`
> **dihapus**. Tipe kontrak bersama — `CustomINode`, `Props` komponen, dan tipe
> domain data (`INativeAds`, dll.) — pindah ke `contracts`, sink tipe yang
> di-import semua layer. Ini juga menghapus `base-ui → service` sepenuhnya.

**Alasan**: Aturan ownership yang tegas adalah prasyarat reuse. Selama
orkestrasi boleh tinggal di app, ia akan selalu dicopy-paste saat app baru
lahir. Menariknya ownership keluar dari app membuat "tambah publisher" = tambah
shell, bukan tambah logika.

**Alternatif yang dipertimbangkan**: Membiarkan orkestrasi di app tapi
dipindahkan ke util bersama yang diimport tiap app. Ditolak: util bersama tetap
dipanggil dari app dengan glue-code per app yang masih bisa drift (persis yang
terjadi sekarang), dan tanpa boundary (AD-7) tidak ada yang mencegah tiap app
menambah cabang lokal.

---

## AD-2: Lokasi orkestrasi halaman — **page loader** di `view-model`

**Konteks**: Situs ini publisher SEO-critical; data harus utuh di initial HTML
(**SSR**), jadi transform layout dijalankan di dalam page (server) supaya client
menerima data yang sudah jadi — pola yang sudah ada di
`libs/view-model/src/lib/layout/transform.ts` dan
`libs/view-model/src/lib/detail-article/entities/transformArticlePageData.ts`.
Pertanyaannya: di mana "perekat" (fetch + rakit opsi + panggil transform)
sebaiknya tinggal?

**Keputusan**: Perekat itu jadi fungsi **page loader** yang tinggal di
`view-model` (server-only), satu per jenis halaman. Contoh:
`loadCategoryPage(input)` mengembalikan data siap-render (`CategoryPageResult`)
atau sinyal `{ notFound: true }` (lihat kontrak `LoadCategoryOutput` di
`02-technical-design.md`). App memanggilnya; app **tidak** lagi merakit opsi
transform sendiri.

**Alasan**:
- Satu sumber kebenaran → drift antar app hilang (analytics, breadcrumbs, meta
  otomatis konsisten).
- `view-model` memang layer yang menjembatani data mentah → props UI; loader
  adalah bentuk paling eksplisit dari peran itu.
- Pure & server-only → mudah ditest dan mudah dipasangi caching (AD-4).
- Konsisten dengan pola `entities/` yang sudah kalian tetapkan di
  `refactor-detail-article`.

**Alternatif yang dipertimbangkan**: (a) Transform di client — **ditolak keras**,
melanggar requirement SSR/SEO (lihat AD-3). (b) Loader di `base-ui` — ditolak,
`base-ui` harus tetap presentational & bebas dari fetch/env. (c) Tetap di app —
lihat AD-1.

---

## AD-3: Transform tetap di server (SSR), bukan dipindah ke client

**Konteks**: Salah satu cara "menyelesaikan" CPU spike server adalah memindah
transform ke client (server kirim data mentah, browser yang transform).

**Keputusan**: **Tidak.** Transform layout & artikel tetap dijalankan di server,
hasilnya utuh di initial HTML.

**Alasan**: Requirement Product/SEO — konten harus ada di HTML pertama untuk
crawler & LCP. Memindah ke client mengubah konten jadi client-rendered
(buruk untuk SEO, buruk untuk LCP mobile-dominan Indonesia). Biaya CPU
diselesaikan lewat AD-4 (caching + transform murni), bukan dengan mengorbankan
SSR.

**Alternatif yang dipertimbangkan**: Hybrid (header/LCP-critical di server, sisa
di client) — sudah dibahas & ditolak di `refactor-detail-article` AD-4 (PM
mewajibkan full-page SSR sinkron saat first open). Konsisten dipertahankan di
sini.

---

## AD-4: Strategi biaya CPU — cache hasil transform + transform murni & O(n)

**Konteks**: Custom server Node single-threaded (`apps/*/server/main.ts`).
Saat ini cache hanya di level HTTP header CDN (`stale-while-revalidate` di
`next.config.js`); React `cache()` di page hanya dedupe **dalam satu request**.
Akibatnya tiap request yang lolos ke server menjalankan fetch + transform penuh,
dan transform sinkron yang berat (map headline, `new URL()` per item di
`transformArticlePageData.ts`, rakit objek analytics per item) memblok event
loop → CPU spike saat trafik naik / cache stampede.

**Keputusan**: Bukan satu peluru, tapi tiga lever berlapis, dan **loader (AD-2)
adalah tempat memasangnya sekali** untuk semua app:

1. **Cache hasil transform**, bukan hanya HTTP. Bungkus loader dengan cache
   server (mis. `unstable_cache`/ISR) berkunci `page + publisher + platform
   (+ regional)`, dengan `revalidate` selaras window CDN yang ada. Efek:
   transform berat jalan **sekali per window revalidate**, bukan per request.
2. **Jaga transform tetap murni & O(n), minimalkan alokasi.** Hindari `new URL()`
   per item (precompute host sekali), rakit objek analytics **hanya** saat
   `is_track` aktif, hindari pass ganda pada array. Fungsi murni juga
   prasyarat agar aman di-cache.
3. **Pisahkan varian device dari kunci cache besar.** Struktur konten yang
   mahal dihitung sekali; perbedaan `isMobile` (mis. sidebar disembunyikan)
   diterapkan sebagai adaptasi murah, supaya cache tidak pecah 2x lipat tanpa
   perlu.

**Kontrak eksplisit**: hasil transform yang di-cache **wajib serializable**
(tanpa `ReactNode`/fungsi). Ini sudah sejalan dengan pola `AdSlotPlaceholder`
(ads di-resolve di client), jadi ads tidak menghalangi caching.

**Alasan**: Menyerang akar CPU spike (transform per request) tanpa mengorbankan
SSR (AD-3). Karena dipasang di loader bersama, satu implementasi caching berlaku
lintas app — reuse dan penghematan CPU datang dari refactor yang sama.

**Alternatif yang dipertimbangkan**: (a) Naikkan spek/replika server — menaikkan
biaya infra tanpa menyembuhkan penyebab. (b) Worker thread untuk transform —
kompleksitas tinggi, tak perlu bila hasil sudah di-cache. (c) Hanya andalkan
CDN header (status quo) — terbukti tidak cukup saat cache miss/stampede.

**Catatan validasi**: klaim "CPU turun" **wajib diukur** (lihat
`03-migration-and-validation-plan.md`), bukan diasumsikan.

---

## AD-5: Rendering — satu `SectionRenderer` registry menggantikan ~16 `switch(type)`

**Konteks**: ~16 page component (`CategoryPage`, `Homepage`, `IndexPage`,
`Tagpage`, `AuthorPage`, dst.) masing-masing menulis ulang
`switch (node.type) → <Component>` yang nyaris identik. Menambah satu tipe
section berarti menyentuh `transform`, union `CustomINode`, **dan** tiap page —
gampang lupa & inkonsisten (mis. `poll-section` ada di `Homepage` tapi tidak di
`CategoryPage`).

**Keputusan**: Sediakan satu komponen `SectionRenderer` di `base-ui/pages`
berbasis **registry deklaratif** `type → Component`. Page cukup memanggil
`<SectionRenderer nodes={main} />`. Halaman yang butuh subset tipe melewatkan
daftar tipe yang diizinkan (allowlist), bukan menulis switch sendiri.

**Alasan**: "Tambah tipe baru" jadi = 1 entry di registry. Menghapus ratusan
baris duplikat dan menghilangkan sumber inkonsistensi antar-halaman.

**Alternatif yang dipertimbangkan**: Biarkan switch per page — memberi type
narrowing otomatis dari TS, tapi harganya duplikasi 16x. Registry tetap
type-safe via mapped type (lihat `02-technical-design.md`), jadi keuntungan
narrowing dipertahankan tanpa duplikasi.

---

## AD-6: Perbedaan path antar-app = parameter loader, bukan cabang kode

**Konteks**: base-app pakai `[category]/[subcategory]/[slug]` (3-level),
fortuneidn-app pakai `[category]/[slug]` (2-level). Ini alasan sah kedua app
dipisah, tapi tidak boleh jadi alasan orkestrasi diduplikasi.

**Keputusan**: Struktur folder route **tetap per app** (Next mewajibkannya).
Tapi route file hanya mengekstrak param dari path lalu memanggil loader bersama;
perbedaan depth diekspresikan sebagai **argumen** (mis. `subcategorySlug?`
diisi base, dikosongkan fortuneidn), bukan sebagai logika berbeda.

**Alasan**: Perbedaan nyata antar-app terkurung di satu baris yang eksplisit,
sisa 99% orkestrasi tetap bersama. Route file jadi ~8–10 baris.

**Alternatif yang dipertimbangkan**: Route-config generik yang mendeskripsikan
depth path secara data-driven — ditolak untuk sekarang (over-engineering; baru 2
pola path). Terapkan bila sudah ada ≥3 pola berbeda (rule of three).

---

## AD-7: Enforcement — nyalakan Nx tags + `@nx/enforce-module-boundaries`

**Konteks**: `@nx/enforce-module-boundaries` = `"off"`, semua project
`"tags": []`. Tidak ada yang mencegah `view-model` import `base-ui` (dilarang di
AD-9 Filosofi B), app import internal app lain, atau `service` import `base-ui`.
Tanpa ini, aturan AD-1 hanya janji di dokumen.

**Keputusan**: Beri tag `type:*` tiap project dan aktifkan aturan boundary
sesuai arah dependensi AD-1. Diaktifkan **bertahap**: mulai `"warn"` untuk
mengungkap pelanggaran yang sudah ada, bereskan, lalu naikkan ke `"error"`.

Tag: `apps/* → type:app`, `view-model → type:view-model`, `base-ui → type:ui`,
`service-lib → type:data`, `global-lib → type:util`, `config-ui → type:config`,
`contracts → type:contract`.

**Alasan**: Membuat aturan arsitektur **dipaksa mesin**. Ini fondasi
scalability — mencegah pembusukan layering seiring bertambahnya publisher &
kontributor.

**Alternatif yang dipertimbangkan**: Andalkan code review — tidak scalable,
manusiawi untuk lolos. Depcruise/eslint-plugin-import lain — Nx sudah punya alat
natifnya, tak perlu tool tambahan.

---

## AD-8: Scope & non-scope

**Keputusan**: Refactor ini menetapkan **prinsip + PoC 1 halaman** (`category`),
bukan migrasi serentak semua halaman. Konsolidasi `detail-article-campaign`,
internal AMP, dan penggabungan app **di luar scope** (lihat README).

**Alasan**: Perubahan sebesar ini wajib bertahap (Expand-Migrate-Contract) dan
divalidasi (drift hilang + CPU terukur) sebelum dilebarkan. Pola yang sama sudah
terbukti dipakai di `refactor-detail-article`.

---

## AD-9: Pecah dependency melingkar `base-ui ↔ view-model`

**Konteks**: Saat ini ada saling-import antar dua lib:
- `view-model → base-ui` — `layout/transform.ts` & `transform-types.ts` import
  `CustomINode` + berbagai `Props` dari `@idn/base-ui/pages/types` dan
  `design-system/*`.
- `base-ui → view-model` — `CategoryPage` (dan `Homepage`, `IndexPage`,
  `Tagpage`, `AuthorPage`) import `useAdsInjection` + tipe `TransformedLayout`
  dari `@idn/view-model/layout`.

Ini **cycle** di level lib. Lolos sekarang hanya karena boundaries `off`;
begitu AD-7 dinyalakan, cycle ini ditandai error.

**Skala (audit community/member)**: arah `base-ui → view-model` **sistemik** —
**32 file base-ui meng-import view-model, semuanya value/hook** (0 `import type`),
~15 hook berbeda, tidak hanya di `pages/` tapi juga `design-system/`
(`useSectionViewModel`, `useEvents`, gallery, manage-article, search-modal, dst).
Komponen presentasi "pintar" yang mengonsumsi hook view-model adalah pola **yang
sudah mapan**. Ini menggugurkan asumsi awal "cukup pindah 1 hook".

**Keputusan: Filosofi B (DIPILIH).** Putuskan cycle dengan menghapus arah
**`view-model → base-ui`**, dan **mempertahankan `base-ui → view-model`** (sesuai
pola kode). Konkretnya:

1. **Pindahkan tipe kontrak yang di-import view-model dari base-ui ke
   `contracts`.** Yaitu `CustomINode`, `CustomSectionProps`/`TransformedLayout`,
   dan `Props` komponen yang dirujuk transform (`HeadlineCardProps`,
   `DynamicSectionProps`, `BreadcrumbsProps`, dll.). Setelah pindah, `view-model`
   meng-import tipe ini dari `@idn/contracts`, **bukan** dari `base-ui`; komponen
   `base-ui` meng-import `Props`-nya sendiri dari `contracts`. Cycle putus tanpa
   menyentuh 32 consumer.
2. **`base-ui → view-model` tetap sah** — komponen boleh mengonsumsi hook
   view-model (itu Wajah B, AD-10). Yang **dilarang** kini `view-model → base-ui`.
3. **`useAdsInjection`/`getNativeAdState` tetap pindah ke
   `base-ui/pages/hooks/`** — tapi alasannya **AD-10** (mekanik DOM/rendering =
   base-ui), **bukan** demi memutus cycle. Terpisah & tetap benar di Filosofi B.

Hasil arah: `base-ui → (view-model, contracts, config, util)`;
`view-model → (service, contracts, config, util)` — **tanpa** base-ui;
`contracts → (kosong)`. Acyclic: semua jalur berakhir di `contracts`.

**Keputusan turunan — folder**: hook client level-halaman ditaruh di folder
khusus `pages/hooks/` (bukan disebar di tiap page, bukan di `global-lib` yang
lebih rendah dan tak boleh import `CustomINode`). Alternatif co-locate di dalam
`section-renderer/` ditolak agar ada rumah jelas untuk hook sejenis berikutnya.

**Keputusan turunan — tipe data bersama via lib `contracts` (bagian dari Filosofi B)**:
setelah pindah, hook di base-ui masih butuh tipe data (`INativeAds`,
`IDisplayAds`, dll.) yang saat ini di `@idn/service/layout/type`. Audit
menunjukkan base-ui meng-import **~17 tipe dari `service`** lintas 4 domain
(layout, detail-article, poll, gallery) — jauh melewati rule-of-three. Maka:

- **Buat lib baru `libs/contracts`** (alias `@idn/contracts/*`, tag
  `type:contract`) berisi **tipe domain data bersama** — bentuk data yang
  dirujuk **baik** oleh props UI **maupun** oleh service.
- **`service-lib` dan `base-ui` sama-sama import dari `@idn/contracts`.**
  `service/*/type.ts` menyimpan hanya tipe **envelope API mentah**
  (`IResponseX` dengan `status`/`data`) dan re-export tipe domain dari
  contracts bila perlu kompatibilitas.
- Hasil: `base-ui → service` **hilang total**; arah jadi `base-ui → contracts`
  dan `service → contracts`. Ini **benar-benar** mencegah value-import
  `ui → data` (bukan sekadar konvensi), sekaligus menuntaskan sebagian G-1.

**Batas isi contracts (di bawah Filosofi B)**: contracts menampung **dua kelas
tipe** — (1) **domain/data-model** (ad config, content meta, article/poll shape)
dan (2) **kontrak view** (`CustomINode`, `CustomSectionProps`, `Props` komponen).
Tipe **request/response envelope** (`IResponseX`) tetap di `service-lib`.
contracts = layer tipe murni (tidak import apa pun), jadi sink dependensi
yang aman untuk base-ui **dan** view-model **dan** service.

**Opsi yang ditolak** (izinkan `type:ui → type:data` + konvensi type-only):
`@nx/enforce-module-boundaries` tidak membedakan import type vs value, jadi
opsi itu tak bisa dipaksa mesin — hanya konvensi. Karena kebutuhan sudah jauh
melewati rule-of-three, `contracts` lebih tepat.

**Alasan memilih B (bukan A)**: base-ui "pintar" (mengonsumsi hook view-model)
adalah pola mapan di 32 file. Filosofi A menuntut relokasi ~15 hook / 32 file —
mahal & berisiko. Filosofi B memusatkan effort pada **relokasi tipe** (mekanis,
risiko runtime rendah) dan tetap memutus cycle sehingga AD-7 bisa menyala.

**Biaya jujur B**: `CustomINode` mereferensikan ~30 `Props` komponen; agar bisa
tinggal di `contracts` tanpa `contracts → base-ui`, `Props` itu ikut pindah ke
contracts. Ini migrasi tipe **besar tapi mekanis**. Dikerjakan **bertahap**:
selama transisi, `view-model → base-ui` masih ada dan constraint itu ditahan di
`"warn"`; baru dinaikkan ke `"error"` setelah relokasi tipe selesai (lihat
`03-migration-and-validation-plan.md`).

**Alternatif yang ditolak**: (a) **Filosofi A** (putuskan `base-ui → view-model`,
base-ui "bodoh") — ditolak: menyentuh 32 consumer & melawan pola kode. (b)
Biarkan cycle + kecualikan dari boundaries — ditolak, melubangi enforcement.
(c) Izinkan `type:ui → type:data` + konvensi type-only — ditolak: Nx tak bisa
paksa type-only, dan kebutuhan sudah melewati rule-of-three.

> Jejak keputusan: fork A/B sempat dibuka setelah audit member/community
> menemukan skala 32-file. **Pemilik arsitektur memilih B.** `CustomINode`
> yang semula dianggap "milik base-ui" kini direlokasi ke `contracts` sebagai
> konsekuensi B.

---

## AD-10: Peran & batas `view-model`

**Konteks**: Muncul pertanyaan apakah `view-model` cukup "hanya transform data
API jadi data siap-pakai untuk base-ui", dan apakah namanya masih relevan.
Kode menunjukkan view-model **bukan** hanya transform: ia juga memuat view-state
& behavior client (`detail-article/client/useArticleAnalytics`,
`useNextArticlePagination`, `usePurchaseArticle`, `useQuizHandlers`, dll.) dan
context stateful (`form-article/context.tsx`).

**Keputusan**: Definisikan peran view-model sebagai jembatan Model↔View dengan
**dua wajah, dipisah runtime**, dan pertahankan namanya:

- **Wajah A — Presenter (server)**: `page-loader/` + `entities/`. Fetch (via
  service-lib) + pure transform → data serializable siap-render, di-cache
  (AD-4). Diekspor lewat `view-model/src/server.ts`.
- **Wajah B — ViewModel klasik (client)**: hooks stateful — analytics,
  pagination, purchase flow, quiz, form-state. Diekspor lewat
  `view-model/src/index.ts`.

**Aturan pembeda (untuk memutuskan lokasi hook/logic):**

| Pertanyaan | Ya → |
|---|---|
| Membentuk data / mengorkestrasi behavior & state app (analytics, pagination, flow, form)? | `view-model` |
| Soal bagaimana komponen dirender/berperilaku visual (sisip ads ke node, scroll elemen, DOM)? | `base-ui` |
| Akses data mentah / HTTP ke API? | `service-lib` |

**Nama tetap `view-model`**: karena ia menyimpan state+behavior (bukan sekadar
transform), nama MVVM ini akurat. Bila kelak seluruh Wajah B dikeluarkan
sehingga tinggal transform murni, barulah nama seperti `presenter`/`page-data`
lebih tepat. Rename lintas monorepo sekarang = churn besar, nilai kecil.

**Konsekuensi**: (1) formalkan `src/server.ts` (saat ini alias ada tapi file
belum ada); (2) hal yang melanggar batas ini dicatat di
`04-current-state-gaps.md` untuk dibereskan bertahap.

**Alasan**: Batas peran yang eksplisit + aturan pembeda mencegah view-model
kembali jadi "tempat sampah" campur transform/DOM/fetch. Ini prasyarat agar
AD-1 (ownership) dan AD-7 (boundaries) bisa ditegakkan konsisten.

**Alternatif yang dipertimbangkan**: Sempit view-model jadi transform-only +
pindah semua hook ke base-ui — ditolak: behavior seperti purchase-flow &
pagination adalah logika view/aplikasi, bukan presentasi; menaruhnya di base-ui
mengotori layer presentational dengan state & efek.

# Technical Design — Refactor Arsitektur Monorepo

## Diagram Alur Data (target)

```
  Route file (apps/*, ~8 baris)
     |  ekstrak param dari path + context publisher
     v
  loadCategoryPage(input)            <-- view-model (server-only, di-cache)
     |  fetch (service-lib)  ->  transform (pure, O(n))
     v
  CategoryPageResult (serializable: CustomINode[] + meta)
     |
     v
  <CategoryPage>                     <-- base-ui (presentational)
     |  <SectionRenderer nodes=... /> <-- registry type -> Component
     v
  HTML (SSR, konten utuh)  +  ads di-resolve di client (AdSlotPlaceholder)
```

Prinsip: **satu loader dipakai base & fortuneidn**; perbedaan hanya di argumen
(`subcategorySlug`). Caching dipasang di loader → berlaku untuk semua app.

## Dependency graph (yang dipaksa Nx boundaries)

```
type:app        -> type:view-model, type:ui, type:config, type:util, type:contract
type:view-model -> type:data, type:config, type:util, type:contract   (TANPA type:ui — AD-9 B)
type:ui         -> type:view-model, type:config, type:util, type:contract
type:data       -> type:util, type:contract
type:config     -> type:util
type:util       -> type:contract
type:contract   -> (pure types, tidak import apa pun)

# Filosofi B: base-ui BOLEH konsumsi hook view-model (ui -> view-model);
# view-model TIDAK boleh import base-ui. Semua jalur berakhir di contract (acyclic).
```

## Kontrak Tipe

### Input & output loader

```ts
// libs/view-model/src/lib/page-loader/category/types.ts
import { IPublishers } from '@idn/config-ui/theme';
import { CustomINode } from '@idn/base-ui/pages/types';
import { BreadcrumbsProps } from '@idn/base-ui/design-system/breadcrumbs';
import { PageTitleProps } from '@idn/base-ui/design-system/page-title';

export type LoadCategoryInput = {
  categorySlug: string;
  subcategorySlug?: string;   // base isi; fortuneidn kosongkan (AD-6)
  publisher: IPublishers;
  regional?: string;
  isMobile: boolean;
  domain: string;
};

// Serializable: TIDAK boleh ada ReactNode/fungsi (syarat cache, AD-4).
export type CategoryPageResult = {
  sections: { main: CustomINode[]; sidebar: CustomINode[] };
  meta: {
    title: string;
    description: string;
    keywords?: string;
    canonicalPathname: string;
  };
  titlePage: PageTitleProps;
  breadcrumbs: BreadcrumbsProps;
};

export type LoadCategoryOutput = CategoryPageResult | { notFound: true };
```

### Registry section renderer (type-safe)

```ts
// libs/base-ui/src/lib/pages/section-renderer/registry.tsx
import { FC } from 'react';
import { CustomINode } from '../types';

// Mapped type menjaga narrowing: tiap entry menerima content sesuai type-nya.
type SectionRegistry = {
  [K in CustomINode['type']]?: FC<Extract<CustomINode, { type: K }> extends {
    content: infer C;
  }
    ? C
    : never>;
};
```

## Struktur Folder Final

```
libs/contracts/src/lib/         # BARU — tipe kontrak bersama (AD-9 Filosofi B)
  data/                          # domain data: INativeAds, IDisplayAds, IAdditionalContent, IContentMeta, article/poll/gallery …
  view/                          # kontrak view: CustomINode, CustomSectionProps, Props komponen (HeadlineCardProps, DynamicSectionProps …)
  index.ts
  # alias: @idn/contracts/* -> libs/contracts/src/lib/*
  # view-model, base-ui, service sama-sama import TIPE dari sini
  # → view-model TIDAK lagi import base-ui; base-ui TIDAK lagi import service
  # CustomINode pindah dari base-ui/pages/types → contracts/view (staged)

libs/view-model/src/lib/
  page-loader/                     # BARU — orkestrasi halaman bersama (server)
    category/
      index.ts                     # loadCategoryPage()
      types.ts
      buildCategoryOptions.ts      # rakit titlePage/breadcrumbs/currentPageData
      __tests__/
    home/  tag/  author/  ...       # menyusul (fase Migrate)
  layout/                          # transform (tetap), dirapikan sesuai AD-4
    entities/                      # (retrofit) pure transform, O(n)

libs/base-ui/src/lib/pages/
  section-renderer/                # BARU
    SectionRenderer.tsx
    registry.tsx
    index.ts
  hooks/                           # BARU — dipindah dari view-model (AD-9)
    useAdsInjection.ts             # + getNativeAdState (DOM/ads)
    index.ts
  category-page/CategoryPage.tsx   # dipangkas: pakai <SectionRenderer/>

apps/base-app/src/app/(root)/[category]/page.tsx         # shell tipis
apps/fortuneidn-app/src/app/(root)/[category]/page.tsx   # shell tipis
```

## Contoh Kode Kunci

### Route file (app) — sesudah

```tsx
// apps/base-app/.../[category]/page.tsx
export default async function Page({ params }: RouteInterface) {
  const { category } = await params;
  const ctx = await getPublisherContext();   // publisher, regional, isMobile, domain
  const res = await loadCategoryPage({ categorySlug: category, ...ctx });
  if ('notFound' in res) return handleRedirectOrNotFound();
  return <CategoryPage {...res} isMobile={ctx.isMobile} category={category} />;
}
// fortuneidn identik; base-app subcategory route menambah subcategorySlug.
```

### Loader + cache (view-model)

```ts
// libs/view-model/src/lib/page-loader/category/index.ts  (server-only)
import { unstable_cache } from 'next/cache';

async function loadCategoryPageUncached(
  input: LoadCategoryInput,
): Promise<LoadCategoryOutput> {
  const layout = await getLayoutServerPage({
    page: input.subcategorySlug
      ? `category/${input.categorySlug}/${input.subcategorySlug}`
      : `category/${input.categorySlug}`,
    platform: input.isMobile ? 'mobile' : 'desktop',
    publisher: input.publisher,
    regional: input.regional,
  });
  if (!layout.data) return { notFound: true };

  const opts = buildCategoryOptions(input, layout.data);   // satu sumber kebenaran
  const sections = transformLayoutData(layout.data, opts); // pure, O(n) (AD-4)
  return { sections, meta: /* ... */, titlePage: opts.titlePage, breadcrumbs: opts.breadcrumbs };
}

// Cache hasil transform (AD-4): kunci = page+publisher+platform(+regional).
export function loadCategoryPage(input: LoadCategoryInput) {
  const key = ['category', input.categorySlug, input.subcategorySlug ?? '-',
               input.publisher, input.isMobile ? 'm' : 'd', input.regional ?? '-'];
  return unstable_cache(() => loadCategoryPageUncached(input), key, {
    revalidate: 120,               // selaraskan dgn window CDN di next.config.js
    tags: [`category:${input.categorySlug}`],
  })();
}
```

> Catatan: `unstable_cache` di sini ilustratif. Detail mekanisme cache
> (in-memory LRU vs `unstable_cache` vs cacheHandler eksternal) diputuskan saat
> Expand setelah diukur — yang mengikat adalah **kontrak** (hasil serializable,
> berkunci `page+publisher+platform`, transform jalan sekali per window).

### SectionRenderer (base-ui)

```tsx
// libs/base-ui/src/lib/pages/section-renderer/SectionRenderer.tsx
export function SectionRenderer({
  nodes,
  allow,
}: {
  nodes: (CustomINode | null)[];
  allow?: CustomINode['type'][];   // subset opsional (mis. sidebar)
}) {
  return (
    <>
      {nodes.map((node, i) => {
        if (!node) return null;
        if (allow && !allow.includes(node.type)) return null;
        const Cmp = SECTION_REGISTRY[node.type] as FC<unknown> | undefined;
        return Cmp ? <Cmp key={i} {...(node.content as object)} /> : null;
      })}
    </>
  );
}
```

### Perbaikan transform O(n) (AD-4) — contoh `readAlso`

```ts
// SEBELUM: new URL() dialokasikan per item di dalam .map()
// SESUDAH: precompute host sekali di luar loop
const hostFor = makeRegionalHostResolver(domain);  // hitung parts sekali
const items = article.readmore.map((rm, i) => ({
  // ...
  hrefCategory: `${hostFor(rm.detail.regional?.slug)}/${rm.detail.category.slug}`,
  ...(isTrack ? { analytics: buildAnalytics(rm, i) } : {}), // rakit hanya bila perlu
}));
```

## Safeguard ESLint

- `libs/view-model/src/lib/page-loader/**` = server-only; larang import komponen
  yang membawa `ReactNode` ke hasil (jaga serializability untuk cache).
- Boundaries Nx (AD-7) menegakkan graph di atas; `depConstraints` konkret ada di
  `03-migration-and-validation-plan.md` Fase 3.

# Migration & Validation Plan — Refactor Arsitektur Monorepo

Pola: **Expand → Migrate → Contract**. Tidak ada big-bang. Tiap fase punya
kriteria selesai + rollback. PoC dipagari ke satu halaman (`category`) sebelum
dilebarkan.

---

## Fase 0 — Baseline pengukuran (wajib sebelum sentuh kode)

Klaim "CPU turun" & "drift hilang" harus terukur. Ambil baseline dulu:

- **CPU/latency server**: rata-rata & p95 CPU per instance dan TTFB pada route
  `category` saat cache miss (mis. setelah revalidate), pada trafik puncak.
  Sumber: metrik infra yang sudah ada (APM/Sentry performance/host metrics).
- **Drift inventory**: catat perbedaan perilaku base vs fortuneidn untuk
  `category` (analytics `isTrackEnabled`, `hasParent`, susunan `subCategory`,
  `headers()` sync vs async) sebagai daftar yang harus jadi identik setelah
  refactor.

Kriteria selesai: baseline terdokumentasi (angka + daftar drift).

---

## Fase 1 — Expand (risiko: nihil, tidak menyentuh kode live)

Tambah tanpa menghapus. Kode lama tetap jalan.

1. **Buat `libs/contracts` (AD-9 Filosofi B)**. Dua kelompok tipe:
   - `contracts/data/`: tipe **domain data** (`INativeAds`, `IDisplayAds`,
     `IAdditionalContent`, `IContentMeta`, … — ~17 tipe yang di-import base-ui
     dari service). `service/*/type.ts` **re-export** dari contracts supaya
     import lama tidak putus.
   - `contracts/view/`: **kontrak view** — `CustomINode`, `CustomSectionProps`,
     dan `Props` komponen yang dirujuk transform. Relokasi ini **bertahap**
     (per section-type saat halaman migrasi); `base-ui/pages/types` &
     `design-system/*` re-export dari contracts selama transisi.
   Semua additive; belum mengubah arah import consumer.
2. Buat `libs/base-ui/src/lib/pages/section-renderer/` (`SectionRenderer` +
   `registry`). Isi registry dari **union superset** semua case yang ada di 16
   switch sekarang (audit dulu supaya tidak ada tipe yang hilang).
3. Buat `libs/base-ui/src/lib/pages/hooks/` dan pindahkan `useAdsInjection` +
   `getNativeAdState` dari view-model — alasannya **AD-10** (mekanik DOM =
   base-ui), **bukan** memutus cycle. Di Filosofi B, `base-ui` **tetap boleh**
   import hook view-model lain; yang dihapus adalah arah `view-model → base-ui`
   (via relokasi tipe di langkah 1).
4. Buat `libs/view-model/src/lib/page-loader/category/` (`loadCategoryPage` +
   `buildCategoryOptions`) tanpa dipakai app dulu.
5. Tulis test unit: `buildCategoryOptions` (opsi identik untuk input base &
   fortuneidn), dan snapshot `SectionRenderer` vs switch lama untuk fixture
   layout nyata.
6. Belum memasang cache di loader (biar Fase 2 mengukur transform murni dulu).

Kriteria selesai: lib baru ada + test hijau, `nx build` semua app tetap lolos,
nol perubahan pada route/page live.

---

## Fase 2 — Migrate (satu halaman, dua app sekaligus)

Tujuan: buktikan drift hilang & pola reuse jalan lintas app.

1. `CategoryPage` (base-ui): ganti dua blok `switch(type)` → `<SectionRenderer/>`
   (main + sidebar dengan `allow`). Verifikasi visual via Storybook + spec.
2. Route `category` base-app **dan** fortuneidn-app: pangkas jadi shell tipis
   yang memanggil `loadCategoryPage` (AD-6: base isi `subcategorySlug`,
   fortuneidn tidak). Seragamkan `headers()` ke bentuk async.
3. Pastikan daftar drift Fase 0 kini identik antar app (analytics, breadcrumbs,
   subCategory) — ini output utama fase ini.
4. **Pasang caching di loader (AD-4)** dan ukur ulang.

Kriteria selesai:
- Output HTML `category` base & fortuneidn identik secara perilaku (drift = 0).
- `category/[subcategory]` base tetap benar (depth 3-level tidak rusak).
- Metrik CPU/TTFB pada cache miss **tidak naik**; pada cache hit **turun**
  dibanding baseline Fase 0.

---

## Fase 3 — Contract (kunci arsitektur)

1. Lebarkan pola ke halaman berikutnya (`home`, `tag`, `author`, `search`) —
   satu per satu, masing-masing base + fortuneidn.
2. Setelah semua page memakai `SectionRenderer`, hapus switch lama yang sudah
   tak terpakai.
3. **Nyalakan Nx boundaries (AD-7)**: beri `tags` tiap project, set rule ke
   `"warn"`, bereskan pelanggaran yang muncul, lalu naikkan ke `"error"`.

```jsonc
// .eslintrc.json — @nx/enforce-module-boundaries: ubah "off" -> "warn" -> "error"
"depConstraints": [
  { "sourceTag": "type:app",        "onlyDependOnLibsWithTags": ["type:view-model","type:ui","type:config","type:util","type:contract"] },
  { "sourceTag": "type:view-model", "onlyDependOnLibsWithTags": ["type:data","type:config","type:util","type:contract"] }, // TANPA type:ui (AD-9 B)
  { "sourceTag": "type:ui",         "onlyDependOnLibsWithTags": ["type:view-model","type:config","type:util","type:contract"] }, // ui BOLEH konsumsi hook view-model
  { "sourceTag": "type:data",       "onlyDependOnLibsWithTags": ["type:util","type:contract"] },
  { "sourceTag": "type:config",     "onlyDependOnLibsWithTags": ["type:util"] },
  { "sourceTag": "type:util",       "onlyDependOnLibsWithTags": ["type:contract"] },
  { "sourceTag": "type:contract",   "onlyDependOnLibsWithTags": [] }
]
```

Kriteria selesai: semua halaman target pakai loader + SectionRenderer, switch
lama terhapus, `base-ui` tidak lagi import `service`, **`view-model` tidak lagi
import `base-ui`** (relokasi `CustomINode`/`Props` ke contracts tuntas),
boundaries `"error"` lolos di CI.

> AD-9 **Filosofi B**: `type:ui → type:view-model` **diizinkan** (komponen
> konsumsi hook), `type:view-model → type:ui` **dilarang**, dan `type:ui →
> type:data` hilang (tipe pindah ke `type:contract`). **Selama relokasi tipe
> (`CustomINode` + `Props`) belum tuntas**, tahan constraint `view-model ∤→ ui`
> di `"warn"` dulu; naikkan ke `"error"` setelah relokasi selesai.

---

## Rencana Validasi

### Reuse / drift (fungsional)
- Diff perilaku base vs fortuneidn untuk tiap halaman termigrasi = 0 (bandingkan
  opsi transform yang dirakit + output analytics).
- "Tambah tipe section baru" cukup 1 entry registry + (bila perlu) 1 case
  transform — tidak menyentuh page component mana pun.

### Biaya CPU (infra) — AD-4
- Bandingkan CPU p95 & TTFB route `category` sebelum/sesudah, pada cache miss
  dan cache hit terpisah.
- Konfirmasi transform berat jalan **sekali per window revalidate**, bukan per
  request (mis. via log/trace hit-miss cache).
- Uji beban ringan (mis. autocannon) pada satu instance untuk melihat event loop
  tidak lagi tersaturasi saat burst.

> Tanpa RUM lapangan saat dokumen ini dibuat; angka diambil dari APM/host metrics
> yang sudah berjalan. Bila baseline tidak bisa diambil, minimal ukur di staging
> dengan replay trafik.

---

## Checklist Rollback per Fase

| Fase | Cara rollback |
|---|---|
| 1 Expand | Hapus folder lib baru — nol dampak (belum dipakai) |
| 2 Migrate | Revert route `category` + `CategoryPage` ke switch/orkestrasi lama (satu commit terisolasi) |
| 3 Contract | Boundaries `"error"` → turunkan ke `"warn"`; page yang belum stabil dikembalikan ke switch lama per-file |

# Current-State Gaps — Temuan Salah-Tempat (Audit view-model)

Daftar temuan konkret dari audit isi `libs/view-model` terhadap definisi peran
di AD-10 (Wajah A presenter/server + Wajah B behavior/client; **bukan**
rendering/DOM, **bukan** akses data mentah). Setiap temuan punya bukti file +
severity + arah perbaikan. Dipakai sebagai checklist yang bisa dicicil — bukan
harus selesai sekaligus.

Metode audit: `grep` untuk DOM/`document`, import adapter/axios, `fetch(`,
import komponen base-ui non-type, dan varian transform per-runtime.

---

## G-1 — Data-access bocor ke view-model (harusnya service-lib) — **HIGH**

**Temuan**: 6 file view-model meng-import **adapter axios mentah** dan menyusun
HTTP call sendiri, melompati service-lib.

| File | Bukti |
|---|---|
| `section/index.ts` | `apiInternal.get('/section', ...)` (baris ~47) |
| `detail-article-amp/index.ts` | `import api from '@idn/service/adapter/api'` |
| `header-footer-amp/index.ts` | `import api from '@idn/service/adapter/api'` |
| `header-footer-pages/index.ts` | `import apiInternal from '@idn/service/adapter/api-internal'` |
| `layout-amp/index.ts` | `import api from '@idn/service/adapter/api'` |
| `layout-pages/index.ts` | `import apiInternal from '@idn/service/adapter/api-internal'` |

**Kenapa salah**: penyusunan endpoint + `.get()` adalah tanggung jawab
service-lib (AD-10). Endpoint tercecer di dua layer; jaminan interceptor
(Sentry, sanitasi, header) yang ada di adapter jadi tak konsisten.

**Arah perbaikan**: pindahkan pemanggilan ke fungsi service (mis.
`LayoutService.getLayoutSection`), view-model cukup memanggil fungsi itu.

**Catatan**: mayoritas view-model **sudah** benar lewat `@idn/service/*`
(20+ pemakaian `@idn/service/layout`, dst.); hanya 6 file ini yang menyimpang.

---

## G-2 — Mekanik DOM/rendering di view-model (harusnya base-ui) — **HIGH**

**Temuan**:
- `layout/index.ts` → `getNativeAdState` (`document.getElementById`, `.remove()`,
  `insertBefore`) + `layout/useAdsInjection.ts` — manipulasi DOM & penempatan ads.
- `detail-article/client/useScrollToTop.ts` — scroll DOM elemen (mekanik presentasi).

**Kenapa salah**: beroperasi pada DOM / `CustomINode[]` = concern rendering
(base-ui), bukan orkestrasi data (AD-10).

**Arah perbaikan**: `useAdsInjection` + `getNativeAdState` →
`base-ui/pages/hooks/` (sudah jadi keputusan AD-9). `useScrollToTop` kandidat
ikut. **Pilah per-hook**: `useArticleScrollTracking` / `useArticleAnalytics`
**tetap** view-model (itu behavior analitik, bukan mekanik render).

---

## G-3 — Transform sama diduplikasi per-runtime — **HIGH (inti reuse)**

**Temuan**: transform logis yang sama dipecah jadi god-file terpisah hanya
karena target render berbeda (web / amp / pages):

```
layout/          layout-amp/          layout-pages/
header-footer/   header-footer-amp/   header-footer-pages/
```

**Kenapa salah**: ini persis masalah drift yang jadi motivasi refactor, tapi di
level view-model. Perubahan aturan transform harus disalin ke 3 tempat.

**Arah perbaikan**: **satu core transform** + adapter kecil per-target
(web/amp/pages), bukan 3 file mandiri. Kandidat besar untuk pemangkasan baris.

---

## G-4 — God-file mencampur Wajah A + Wajah B — **MEDIUM**

**Temuan**:

| File | Baris | Campur |
|---|---|---|
| `detail-article-amp/index.ts` | ~1688 | transform + fetch adapter + behavior |
| `detail-article-campaign/index.ts` | ~1207 | ~90% duplikat `detail-article` |
| `community/index.ts` | ~1048 | transform + state + DOM |
| `form-article/context.tsx` | ~1065 | context + JSX + fetch |

**Kenapa salah**: mencampur presenter (server) dan behavior (client) dalam satu
modul membuat batas Wajah A/B tak terlihat + menghalangi caching & test.

**Arah perbaikan**: ikuti pola `detail-article` yang **sudah** dipecah
`entities/` + `client/`. Sebagian sudah tercatat sebagai backlog di
`refactor-detail-article/01-architecture-decisions.md` (AD-1: konsolidasi
`detail-article-campaign`).

---

## G-5 — Split server/client belum diformalkan — **MEDIUM**

**Temuan**: alias `@idn/view-model/server` ada di `tsconfig.base.json`, tapi
file `libs/view-model/src/server.ts` **belum ada**. 22 file `'use client'`,
0 `'use server'`.

**Kenapa salah**: Wajah A (transform server) & Wajah B (hooks client) belum
terpisah di batas modul → kode transform server berisiko ikut ter-bundle ke
client (memberati JS browser tanpa perlu).

**Arah perbaikan**: formalkan `src/server.ts` (ekspor loader + entities) vs
`src/index.ts` (ekspor client hooks), sesuai AD-10.

---

## G-6 — Community-landing men-transform di CLIENT (langgar SSR/AD-3) — **HIGH**

**Temuan**: alur community landing (base-app) berbeda dari `category`:
- `community/page.tsx` (server) fetch layout → oper **data mentah**
  (`dataLayout={landing.data}`) ke `<CommunityLanding>` (app-ui, `'use client'`).
- `useCommunityLandingViewModel` (`view-model/community-landing/index.ts`)
  men-transform di **client** via `useMemo` + `switch(type)` (termasuk
  `new URL()` untuk strip subdomain), baru render lewat `CommunityLandingPage`.

**Kenapa salah**: transform terjadi di browser → section **tidak ada** di initial
HTML (buruk untuk SEO/LCP), dan biaya transform pindah ke tiap klien alih-alih
sekali di server yang bisa di-cache. Ini kebalikan dari keputusan AD-3/AD-4.

**Arah perbaikan**: pindahkan ke **server loader** (`loadCommunityLandingPage`)
seperti `category`; `CommunityLandingPage` pakai `SectionRenderer`.

---

## G-7 — base-ui → view-model sistemik (32 file), termasuk member-hub — **HIGH**

**Temuan**: cycle `base-ui ↔ view-model` (AD-9) **jauh lebih luas** dari dugaan
awal. Audit: **32 file base-ui meng-import view-model**, **semuanya value import
(hook), bukan type** — ~15 hook berbeda:

| Contoh consumer base-ui | Hook view-model |
|---|---|
| `pages/member-hub-page/useMemberProfileHub.tsx` | `useMemberProfile` |
| `pages/member-hub-page/MemberHubPage.tsx`, `unlocked-article-page` | `useUnlockedArticles` |
| `design-system/dynamic-section/*` | `useSectionViewModel` |
| `design-system/event/Event.tsx` | `useEvents` |
| `design-system/gallery/*`, `manage-article`, `search-modal`, `filter-section`, `notification`, `article-paywall` | hook masing-masing |
| `pages/{category,home,index,tag,author}` | `useAdsInjection` (×6) |

Contoh mencolok: `useMemberProfileHub` adalah **hook view-model yang tinggal di
base-ui** dan membungkus `useMemberProfile` (view-model) — pelanggaran ganda
(base-ui berisi behavior + import view-model).

**Kenapa penting**: premis AD-9 ("murah, pindah 1 hook") terbantah. Memutus cycle
ini adalah **workstream besar**, dan arah pemutusannya (Filosofi A vs B) kini
menjadi keputusan terbuka — lihat **revisi skala di AD-9**.

---

## Divergensi produk: community base vs fortuneidn (BUKAN gap, catatan)

`community/page.tsx` base = landing native (fetch layout + transform), sedangkan
fortuneidn = **iframe** ke URL community eksternal (`formatUrlIframe`, render
`<CommunityPage src=... />`). Ini **perbedaan produk nyata**, bukan drift —
**jangan** dipaksa ke satu loader. Loader hanya berlaku untuk varian native.

> Catatan keamanan minor: `formatUrlIframe` di fortuneidn menaruh
> `access_token=123-token&source=123-source` — tampak placeholder ter-hardcode.
> Perlu dicek apakah ini nilai asli/rahasia yang bocor. Bukan bagian arsitektur,
> tapi layak ditinjau terpisah.

---

## G-8 — God-file `community` + render-switch community/member — **MEDIUM**

**Temuan**: `view-model/community/index.ts` (~1048 baris) memuat `switch(type)`
transform + state. `CommunityPage` (base-ui, 11 case), `CommunityLandingPage`
(4 case), `MemberHubPage` (4 case) — render-switch yang sama, kandidat
`SectionRenderer` (AD-5).

**Arah perbaikan**: sama dengan G-3/G-4 — pisah `entities/`+`client/`, pakai
`SectionRenderer`. `useMemberProfileHub` pindah dari base-ui ke `view-model`
(behavior = Wajah B, AD-10).

---

## Yang sudah benar (jangan diutak-atik)

- Mayoritas fetch sudah lewat `@idn/service/*` (lihat catatan G-1).
- Pola `entities/` + `client/` di `detail-article` — template target untuk lib lain.

## Catatan relokasi tipe ke `contracts` (AD-9 Filosofi B)

Dua arah import tipe yang harus pindah ke `@idn/contracts` (bukan gap terpisah,
sudah diputuskan di AD-9):
- **`base-ui → service`** (~17 tipe: `INativeAds`, `IDisplayAds`, dll.) →
  `contracts/data/`.
- **`view-model → base-ui`** (`CustomINode`, `Props` komponen, dan
  `IContent` dari `design-system/article-content` yang dipakai
  `detail-article/entities/format*`) → `contracts/view/`.

Keduanya membuat `contracts` jadi sink tipe, memutus cycle (Filosofi B) dan
menghapus `base-ui → service`. Dikerjakan bertahap mulai Fase 1 (Expand), lihat
`03-migration-and-validation-plan.md`.

---

## Ringkasan prioritas

| ID | Temuan | Severity | Perbaikan mengarah ke |
|---|---|---|---|
| G-1 | Data-access bocor | HIGH | service-lib |
| G-2 | DOM/ads di view-model | HIGH | base-ui (AD-9) |
| G-3 | Transform per-runtime ×3 | HIGH | satu core + adapter |
| G-4 | God-file campur A+B | MEDIUM | entities/ + client/ |
| G-5 | server/client belum formal | MEDIUM | `server.ts` vs `index.ts` |
| G-6 | Community-landing transform di client | HIGH | server loader (AD-3/AD-4) |
| G-7 | base-ui → view-model sistemik (32 file) | HIGH | keputusan Filosofi A/B (AD-9 revisi) |
| G-8 | God-file community + render-switch | MEDIUM | entities/ + `SectionRenderer` |

Urutan eksekusi disarankan: G-1 & G-2 dulu (membersihkan identitas view-model),
lalu G-3 (dampak reuse terbesar), G-4/G-5 menyusul bersama migrasi per-halaman.
**G-7 memblok penyalaan boundaries (AD-7)** — keputusan Filosofi A/B di AD-9 harus
diambil lebih dulu. G-6/G-8 ikut saat migrasi community/member (lihat
`03-migration-and-validation-plan.md`).

# Refactor Arsitektur Monorepo — Scalability & Reuse Lintas App

## Tujuan

Membuat arsitektur monorepo IDN **scalable** dan **reusable lintas app** —
sehingga menambah publisher/app baru menjadi murah, konsisten, dan tidak
menimbulkan duplikasi yang drift. Fokus mencakup empat layer bersama:
**UI** (`base-ui`), **service** (`service-lib`), **utils** (`global-lib`),
dan **view-model** (`view-model`).

Tiga sasaran sekaligus:

1. **Reuse** — orkestrasi halaman (fetch → rakit opsi → transform → render)
   jadi satu sumber kebenaran di lib, bukan dicopy-paste per app.
2. **Scalability** — batas antar-layer dipaksa mesin (Nx module boundaries),
   bukan sekadar janji. Publisher ke-N = shell tipis + config, nol logika baru.
3. **Biaya infra (CPU)** — transform tetap di server (SSR, wajib demi SEO),
   tapi biayanya dibatasi lewat caching hasil transform + transform yang murni
   dan O(n). Ini menjawab CPU spike yang pernah terjadi.

## Konteks & Bukti Pemicu

Refactor ini bukan estetika. Ada dua bukti konkret dari kode saat ini:

- **App layer terlalu tebal & sudah drift.** `apps/base-app/.../[category]/page.tsx`
  dan `apps/fortuneidn-app/.../[category]/page.tsx` ~85% identik tapi menyimpang
  diam-diam: satu pakai `headers()?.get()` (sinkron, Next lama) satu
  `(await headers()).get()` (async); satu kirim `isTrackEnabled: true` +
  `hasParent`, satu tidak (analytics & perilaku beda **tanpa disengaja**).
  Setiap halaman dicopy-paste per app lalu berevolusi sendiri.
- **Tidak ada enforcement boundary.** `@nx/enforce-module-boundaries` = `"off"`,
  semua project `"tags": []`. Apa pun boleh import apa pun — layering pasti
  membusuk seiring pertumbuhan.

Split app **tetap dipertahankan** (bukan digabung): base vs fortuneidn punya
struktur path detail-article berbeda (`[category]/[subcategory]/[slug]` 3-level
vs `[category]/[slug]` 2-level), dan varian AMP wajib Pages Router terpisah
karena App Router tidak lagi mendukung AMP. Fokus refactor = **maksimalkan
reuse lintas app**, bukan mengurangi jumlah app.

## Struktur Dokumen

| File | Isi |
|---|---|
| `README.md` | Dokumen ini — ringkasan, konteks, index |
| `01-architecture-decisions.md` | Keputusan fundamental (ADR-style) AD-1 s/d AD-10: ownership layer, lokasi orkestrasi, strategi biaya CPU, section renderer, module boundaries, pecah cycle base-ui↔view-model (AD-9), peran & batas view-model (AD-10) |
| `02-technical-design.md` | Kontrak tipe, struktur folder, diagram alur data, desain cache, contoh kode konkret |
| `03-migration-and-validation-plan.md` | Rencana Expand-Migrate-Contract + cara validasi reuse & CPU |
| `04-current-state-gaps.md` | Temuan audit view-model saat ini (G-1 s/d G-5): data-access bocor, DOM di view-model, transform ×3 per-runtime, god-file, split server/client belum formal |

## Scope

**Termasuk:**
- Prinsip layering + ownership antar `apps` / `libs` (AD-1)
- Lapisan **page loader** bersama di `view-model` (AD-2)
- Transform tetap SSR + strategi caching hasil transform untuk batasi CPU (AD-3, AD-4)
- **Section renderer registry** menggantikan ~16 `switch(type)` (AD-5)
- Perbedaan path antar-app sebagai parameter, bukan cabang kode (AD-6)
- **Nx tags + `@nx/enforce-module-boundaries`** dinyalakan bertahap (AD-7)
- **Pecah cycle `base-ui ↔ view-model` via Filosofi B** (AD-9): hapus
  `view-model → base-ui`, pertahankan `base-ui → view-model`
- **Lib baru `@idn/contracts`** untuk tipe kontrak bersama — domain data
  **+ kontrak view** (`CustomINode`, `Props`); menghapus `base-ui → service`
  **dan** `view-model → base-ui`
- **Definisi peran & batas `view-model`** (AD-10)

**Di luar scope (backlog terpisah):**
- Konsolidasi `detail-article-campaign` → `detail-article` (~90% duplikat) —
  sudah tercatat di `refactor-detail-article/01-architecture-decisions.md` (AD-1).
- Menyentuh internal AMP (`*-pages`) — constraint berbeda, tidak diprioritaskan.
- Menggabungkan app (ditolak, lihat konteks di atas).

## Status

Draft awal untuk didiskusikan. Belum ada kode yang disentuh. Setelah keputusan
fundamental (AD-1 s/d AD-10) disepakati, masuk fase eksekusi — dimulai dari
**Fase 0 (baseline pengukuran CPU + inventory drift)**, lalu Expand, dengan PoC
di satu halaman (`category`) untuk membuktikan drift hilang & biaya CPU turun
sebelum dilebarkan.

Tidak ada data RUM/pengukuran CPU lapangan yang dilampirkan saat dokumen ini
dibuat — keputusan berbasis analisis kode + prinsip. Lihat
`03-migration-and-validation-plan.md` untuk cara mengukur setelah implementasi.
