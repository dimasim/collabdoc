# 🚀 Milestone 1 — Setup Awal CollabDoc

> **Tujuan:** Kamu punya struktur folder monorepo yang rapi, backend NestJS bisa jalan & koneksi ke PostgreSQL via Prisma, dan frontend Next.js bisa running. Belum ada fitur, tapi fondasi sudah benar.

---

## Mengapa Setiap Langkah Penting?

Sebelum mulai, pahami **apa yang kita bangun**:

```
[Browser / Next.js]  <──REST──>  [NestJS API]  <──Prisma──>  [PostgreSQL]
```

- **Next.js** = UI yang user lihat di browser
- **NestJS** = "otak" server: terima request, proses logika, simpan/ambil data
- **Prisma** = jembatan antara NestJS dan database (query DB pakai TypeScript, bukan SQL mentah)
- **PostgreSQL** = tempat data disimpan secara permanen

---

## Prasyarat

Pastikan sudah terinstall:

```bash
node --version      # >= 18.x (dibutuhkan oleh beberapa tooling)
psql --version      # PostgreSQL client
```

> **Catatan:** Jika PostgreSQL berjalan via Podman, pastikan container sudah up sebelum mulai.

---

## Langkah 1 — Inisialisasi Struktur Monorepo

### Apa itu monorepo?

Monorepo = satu repository Git yang berisi **lebih dari satu project** (frontend + backend).
Keuntungannya: mudah share tipe TypeScript, satu git history, satu tempat untuk semua kode.

```bash
# Kamu sudah berada di: d:\belajar\collabdoc\collabdoc\
mkdir -p apps/web apps/server
```

Struktur yang kita tuju:

```
collabdoc/
├── apps/
│   ├── web/        <- Next.js (frontend)
│   └── server/     <- NestJS (backend)
├── collaborative-editor-spec.md
└── README.md
```

---

## Langkah 2 — Setup NestJS Backend

### Kenapa NestJS?

NestJS adalah framework Node.js yang menggunakan **dekorator** (mirip anotasi di Java/Spring).
Ia memaksa kita menulis kode yang **terstruktur dan modular** — setiap fitur punya module,
controller, dan service sendiri. Ini yang membuat codebase mudah dikelola saat project membesar.

### Konsep Inti NestJS (penting dipahami dulu!):

| Komponen | Fungsi | Analogi |
|---|---|---|
| **Module** | Mengelompokkan fitur terkait | Folder/namespace |
| **Controller** | Menerima HTTP request, return response | Router di Express |
| **Service** | Berisi business logic | Helper/utility functions |
| **DTO** | Mendefinisikan shape data yang masuk | Interface untuk request body |

### Install dan scaffold project:

```bash
cd apps/server
npx @nestjs/cli new . --package-manager npm --skip-git --strict
```

> `--strict` mengaktifkan TypeScript strict mode — best practice, hindari `any` sedini mungkin.

Setelah selesai, struktur `apps/server/src/` akan terlihat:

```
src/
├── app.controller.ts    <- controller contoh (boleh dihapus nanti)
├── app.module.ts        <- root module (entry point semua module)
├── app.service.ts       <- service contoh (boleh dihapus nanti)
└── main.ts              <- bootstrap aplikasi NestJS
```

### Verifikasi NestJS berjalan:

```bash
npm run start:dev
```

Buka `http://localhost:3000` -> harus muncul `"Hello World!"`.

### Ubah port ke 4000

Edit `apps/server/src/main.ts`:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.setGlobalPrefix('api'); // semua route jadi /api/...
  await app.listen(4000);
  console.log('Server running on http://localhost:4000');
}
bootstrap();
```

> Kita pakai port 4000 agar tidak bentrok dengan Next.js di port 3000.
> `setGlobalPrefix('api')` membuat semua endpoint otomatis prefix `/api/`.

---

## Langkah 3 — Setup Prisma + Koneksi PostgreSQL

### Kenapa Prisma?

Prisma adalah **ORM** (Object-Relational Mapper). Daripada nulis SQL mentah:

```sql
SELECT * FROM users WHERE email = 'test@example.com';
```

Kamu cukup nulis TypeScript yang typesafe:

```typescript
const user = await prisma.user.findUnique({ where: { email } });
```

Prisma juga generate tipe TypeScript otomatis dari schema — kalau typo nama field, error saat
compile, bukan saat runtime di production.

### Install Prisma:

```bash
# Dari dalam apps/server/
npm install @prisma/client
npm install -D prisma
```

### Inisialisasi Prisma:

```bash
npx prisma init
```

Perintah ini membuat:
- `prisma/schema.prisma` -> definisi model database
- `.env` -> file environment variables

### Konfigurasi Database URL:

Edit file `apps/server/.env`:

```env
DATABASE_URL="postgresql://user:password@localhost:5432/collabdoc"
JWT_SECRET="ganti-dengan-string-random-panjang-minimal-32-karakter"
PORT=4000
```

> Ganti `user` dan `password` sesuai konfigurasi PostgreSQL kamu.

### Definisikan Schema:

Edit `apps/server/prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String     @id @default(cuid())
  email     String     @unique
  password  String
  name      String?
  documents Document[] @relation("OwnedDocuments")
  createdAt DateTime   @default(now())
}

model Document {
  id        String   @id @default(cuid())
  title     String
  content   String   @default("")
  ownerId   String
  owner     User     @relation("OwnedDocuments", fields: [ownerId], references: [id])
  updatedAt DateTime @updatedAt
  createdAt DateTime @default(now())
}
```

### Jalankan migrasi pertama:

```bash
npx prisma migrate dev --name init
```

Perintah ini:
1. Membaca `schema.prisma`
2. Generate SQL `CREATE TABLE ...`
3. Menjalankan SQL ke PostgreSQL
4. Generate TypeScript client di `node_modules/@prisma/client`

### Verifikasi via Prisma Studio:

```bash
npx prisma studio
```

Buka `http://localhost:5555` -> kamu bisa lihat tabel `User` dan `Document` sudah terbuat.

---

## Langkah 4 — Buat PrismaModule (Best Practice NestJS)

Di NestJS, kita bungkus Prisma ke dalam sebuah **Module** agar bisa di-inject ke module lain
(Auth, Documents, dll). Ini prinsip **Dependency Injection** — NestJS yang urus lifecycle-nya.

### Buat file `apps/server/src/prisma/prisma.service.ts`:

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect(); // connect ke DB saat module diload oleh NestJS
  }
}
```

### Buat file `apps/server/src/prisma/prisma.module.ts`:

```typescript
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global() // @Global() = PrismaService tersedia di semua module tanpa import manual
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

### Daftarkan ke AppModule:

Edit `apps/server/src/app.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { PrismaModule } from './prisma/prisma.module';

@Module({
  imports: [PrismaModule],
})
export class AppModule {}
```

### Verifikasi:

```bash
npm run start:dev
```

Jika tidak ada error koneksi di terminal, Prisma berhasil connect ke PostgreSQL!

---

## Langkah 5 — Setup Next.js Frontend

```bash
# Dari root project
cd apps/web
npx create-next-app@latest . --typescript --tailwind --app --no-src-dir
```

### Konfigurasi environment:

Buat file `apps/web/.env.local`:

```env
NEXT_PUBLIC_API_URL=http://localhost:4000
NEXT_PUBLIC_SOCKET_URL=http://localhost:4000
```

### Verifikasi:

```bash
npm run dev
```

Buka `http://localhost:3000` -> halaman default Next.js harus tampil.

---

## Langkah 6 — Smoke Test End-to-End

Jalankan keduanya bersamaan:

**Terminal 1 (Backend):**

```bash
cd apps/server && npm run start:dev
```

**Terminal 2 (Frontend):**

```bash
cd apps/web && npm run dev
```

### Checklist verifikasi milestone 1:

- [ ] `http://localhost:4000/api` -> NestJS merespons
- [ ] `http://localhost:3000` -> Next.js homepage tampil
- [ ] `http://localhost:5555` (Prisma Studio) -> Tabel `User` dan `Document` ada dan kosong
- [ ] Tidak ada error koneksi database di terminal NestJS

---

## Struktur Akhir yang Diharapkan

```
collabdoc/
├── apps/
│   ├── web/                    [OK] Next.js 14/15 + Tailwind
│   │   ├── app/
│   │   └── package.json
│   └── server/                 [OK] NestJS + Prisma
│       ├── src/
│       │   ├── prisma/
│       │   │   ├── prisma.module.ts
│       │   │   └── prisma.service.ts
│       │   ├── app.module.ts
│       │   └── main.ts
│       ├── prisma/
│       │   └── schema.prisma   [OK] Sudah migrate ke PostgreSQL
│       └── package.json
├── collaborative-editor-spec.md
└── MILESTONE_1_SETUP.md        <- file ini
```

---

## Troubleshooting Umum

| Error | Penyebab | Solusi |
|---|---|---|
| `P1001: Can't reach database server` | PostgreSQL tidak jalan / salah port | Pastikan container Podman/service PostgreSQL up |
| `EADDRINUSE :::3000` | Port sudah dipakai proses lain | Hentikan proses lain, atau ganti port |
| `Module not found: @prisma/client` | Belum generate client | Jalankan `npx prisma generate` |
| `Table 'User' does not exist` | Belum migrasi | Jalankan `npx prisma migrate dev --name init` |
| `Cannot find module './app.module'` | Typo import atau file belum ada | Cek path import di `main.ts` |

---

## Selanjutnya: Milestone 2 — Auth (Register & Login)

Setelah milestone ini selesai, kamu siap masuk ke:

1. Membuat `AuthModule` dengan `POST /api/auth/register` dan `POST /api/auth/login`
2. Password hashing dengan `bcrypt`
3. Generate & validasi JWT (access token + refresh token)
4. `JwtAuthGuard` NestJS untuk protect route

> **Tips:** Selesaikan satu milestone secara tuntas sebelum lanjut. Fondasi yang kuat = milestone
> berikutnya jauh lebih mudah dan tidak perlu refactor besar-besaran.

