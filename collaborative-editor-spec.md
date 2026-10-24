# Real-Time Collaborative Editor — Project Spec

> File ini dibuat untuk jadi context/instruksi pengembangan di AI IDE (Antigravity). Berisi scope, arsitektur, data model, dan milestone supaya AI agent bisa langsung generate/scaffold project sesuai rencana.

## 1. Ringkasan Project

Aplikasi web mirip Google Docs versi sederhana: beberapa user bisa membuka dokumen yang sama dan melihat perubahan teks secara real-time. Fokus utama portofolio: real-time sync (WebSocket), state management, dan arsitektur full stack yang rapi — bukan fitur selengkap Google Docs.

**Nama project (sementara):** CollabDoc

## 2. Tech Stack

| Layer | Teknologi | Alasan & Relevansi Loker (Indonesia) |
|---|---|---|
| Frontend | Next.js 14/15 (App Router) + TypeScript | Standard teratas lowongan Frontend / Fullstack di Indonesia |
| Styling | Tailwind CSS | Cepat, modern, paling populer di startup ID |
| Runtime & PM | Node.js 20 LTS + npm | Stabil, kompatibel penuh dengan ekosistem NestJS |
| Backend | NestJS + TypeScript | High demand di enterprise/fintech ID (Clean Architecture & Modular) |
| Realtime | Socket.io (`@nestjs/websockets`) | WebSocket gateway bawaan NestJS |
| Database | PostgreSQL (via Prisma ORM) | Database relasional paling umum di tech company ID |
| Auth | JWT Manual (Access & Refresh Token) | Konsep auth yang paling sering diuji dalam technical interview |
| Deployment | Podman + Podman Compose (di Server Ini) | Enterprise-ready container deployment (rootless & Docker-compatible) |

**Opsional (fase lanjutan):** Yjs + y-websocket untuk proper CRDT conflict resolution.

## 3. Scope Fitur

### MVP (harus ada)
- [ ] Autentikasi user (register/login)
- [ ] CRUD dokumen (create, list, rename, delete)
- [ ] Buka dokumen → textarea/editor yang bisa diedit
- [ ] Perubahan teks tersinkron real-time ke semua user yang buka dokumen sama
- [ ] Indikator siapa saja yang sedang online di dokumen tersebut

### Fase 2 (nice to have)
- [ ] Cursor presence (lihat posisi kursor user lain dengan warna berbeda)
- [ ] Rich text editor (bold/italic/heading) pakai Tiptap atau Slate
- [ ] Conflict resolution proper pakai Yjs (CRDT) menggantikan sync manual
- [ ] History/version log perubahan dokumen
- [ ] Share dokumen dengan permission (view-only vs edit)

### Di luar scope (jangan overbuild)
- Comment/anotasi
- Export ke PDF/Word
- Mobile app

## 4. Arsitektur

```mermaid
flowchart LR
    subgraph Client
        A[Next.js App]
    end
    subgraph Server
        B[NestJS API]
        C[Socket.io Gateway]
    end
    D[(PostgreSQL)]

    A -- REST: auth, CRUD dokumen --> B
    A <-- WebSocket: sync teks realtime --> C
    B --> D
    C --> D
```

Alur singkat:
1. User login → dapat JWT/session.
2. User buka dokumen → frontend fetch isi dokumen via REST, lalu connect ke Socket.io room dengan `documentId` sebagai room name.
3. Setiap ketikan di client → emit event `doc:change` ke server → server broadcast ke semua client lain di room yang sama → server juga debounce-save ke database.

## 5. Struktur Folder (usulan)

```
collabdoc/
├── apps/
│   ├── web/                 # Next.js frontend
│   │   ├── app/
│   │   │   ├── (auth)/login/
│   │   │   ├── (auth)/register/
│   │   │   ├── documents/
│   │   │   └── documents/[id]/
│   │   ├── components/
│   │   │   ├── Editor.tsx
│   │   │   └── PresenceBar.tsx
│   │   └── lib/socket.ts
│   └── server/               # NestJS + Socket.io backend
│       ├── src/
│       │   ├── auth/
│       │   │   ├── auth.module.ts
│       │   │   ├── auth.controller.ts
│       │   │   ├── auth.service.ts
│       │   │   └── dto/
│       │   ├── documents/
│       │   │   ├── documents.module.ts
│       │   │   ├── documents.controller.ts
│       │   │   ├── documents.service.ts
│       │   │   └── dto/
│       │   ├── gateways/
│       │   │   └── document.gateway.ts
│       │   ├── prisma/
│       │   │   ├── prisma.module.ts
│       │   │   └── prisma.service.ts
│       │   └── app.module.ts
│       ├── prisma/
│       │   └── schema.prisma
│       └── package.json
└── README.md
```

## 6. Data Model (Prisma schema draft)

```prisma
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

## 7. REST API Endpoints

| Method | Endpoint | Deskripsi |
|---|---|---|
| POST | `/api/auth/register` | Registrasi user baru |
| POST | `/api/auth/login` | Login, return token/session |
| GET | `/api/documents` | List dokumen milik user |
| POST | `/api/documents` | Buat dokumen baru |
| GET | `/api/documents/:id` | Ambil isi dokumen |
| PATCH | `/api/documents/:id` | Update title/metadata |
| DELETE | `/api/documents/:id` | Hapus dokumen |

## 8. WebSocket Events

| Event | Arah | Payload | Keterangan |
|---|---|---|---|
| `join-document` | client → server | `{ documentId, userId }` | Join room saat buka dokumen |
| `doc:change` | client ↔ server | `{ documentId, content, userId }` | Broadcast perubahan teks |
| `presence:update` | server → client | `{ users: [...] }` | Update daftar user online di room |
| `leave-document` | client → server | `{ documentId }` | Saat user tutup dokumen/disconnect |

Catatan implementasi: server sebaiknya debounce penyimpanan ke database (misal setiap 1-2 detik idle), bukan setiap keystroke, supaya tidak membebani DB.

## 9. Milestone Pengembangan

1. **Setup awal** — init monorepo, setup Next.js + NestJS + Prisma, koneksi ke PostgreSQL.
2. **Auth** — register/login, protect routes.
3. **CRUD dokumen (belum realtime)** — bisa create/list/edit/delete dokumen biasa via REST.
4. **Realtime sync dasar** — integrasi Socket.io, broadcast perubahan teks antar client di room yang sama.
5. **Presence indicator** — tampilkan siapa saja yang online di dokumen tsb.
6. **Polish & deploy** — deploy frontend ke Vercel, backend ke Railway/Fly.io, testing multi-tab/multi-user.
7. *(Opsional lanjutan)* — ganti sync manual dengan Yjs untuk conflict resolution proper, tambah cursor presence & rich text.

## 10. Environment Variables

```
# apps/server/.env
DATABASE_URL=postgresql://user:password@localhost:5432/collabdoc
JWT_SECRET=
PORT=4000

# apps/web/.env.local
NEXT_PUBLIC_API_URL=http://localhost:4000
NEXT_PUBLIC_SOCKET_URL=http://localhost:4000
```

## 11. Catatan Deployment (Local Server via Podman)

- **Podman & Podman Compose**: Aplikasi dideploy langsung di server ini menggunakan `podman-compose` / `podman play kube`.
- **Services in Container**:
  - `postgres`: Container database PostgreSQL 16.
  - `server`: Container NestJS + Node.js (Backend API & Socket.io Gateway).
  - `web`: Container Next.js + Node.js (Frontend App).
- **Network**: Podman internal network antar container untuk keamanan.
- **Port Mapping**:
  - Web UI: `http://localhost:3000` (atau port server yang ditentukan).
  - API / Socket: `http://localhost:4000`.
