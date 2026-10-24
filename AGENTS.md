# Project Rules & Guidelines — CollabDoc

Dokumen ini berisi panduan dan aturan pengodean (coding guidelines) untuk AI Agent (Antigravity) saat mengedit atau men-generate kode dalam proyek CollabDoc.

---

## 1. Stack & Runtime Rules

- **Package Manager & Runtime**: Gunakan `npm` dan `Node.js` untuk instalasi package dan running script (`npm install`, `npm run dev`, `npx prisma`).
- **Backend**: NestJS dengan TypeScript murni, arsitektur modular (`.module.ts`, `.controller.ts`, `.service.ts`), dan Gateway WebSocket NestJS (`@nestjs/websockets`).
- **Frontend**: Next.js 14/15 (App Router) + TypeScript + Tailwind CSS. Komponen real-time wajib menggunakan directive `'use client'`.
- **Database**: PostgreSQL diakses via Prisma ORM (`prisma/schema.prisma`).
- **Deployment**: Podman & Podman Compose (`podman-compose.yml`).

---

## 2. API & Data Formatting Rules

- **Response Format**: Semua REST API endpoint wajib mengembalikan struktur JSON konsisten:
  ```json
  {
    "success": true,
    "data": { ... },
    "message": "Pesan opsional"
  }
  ```
- **Error Handling**: Gunakan NestJS Built-in Exceptions (`HttpException`, `BadRequestException`, `UnauthorizedException`) untuk error API.
- **Auth Flow**: Gunakan JWT Manual (Access Token & Refresh Token). Token disimpan secara aman dalam Cookie (`HttpOnly`, `SameSite`, `Secure` di production).

---

## 3. Real-Time & WebSocket Rules

- **Namespace & Room**: Setiap dokumen yang dibuka user masuk ke room Socket.io berbasis `documentId`.
- **Debounced DB Persistence**: Jangan lakukan save ke database PostgreSQL pada setiap keystroke. Lakukan debounce save (misal: simpan ke DB setelah 1.5 - 2 detik idle).
- **Presence**: Update daftar user online (`presence:update`) setiap kali ada socket client yang `join-document` atau `disconnect`.

---

## 4. Code Quality & Standards

- **Strict TypeScript**: Hindari tipe data `any`. Buat interface / DTO eksplisit untuk request body, response, dan socket payload.
- **Clean Architecture**: Pisahkan business logic di Service (`.service.ts`), routing/request handler di Controller (`.controller.ts`), dan socket events di Gateway (`.gateway.ts`).
