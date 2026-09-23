# Database Schema for Users Roles

This README documents what was actually done in the workspace after running
**Prompt A** of the Auth & Roles lesson (adding the `technician` /
`supervisor` / `admin` roles and gating work-order actions by who is logged
in). It's written for students following the course so you can see, step by
step, how a single scoped prompt turned into real schema, code, and
verification — not just a description of what *should* happen.

Prompt A only covers the **database + seed data** layer. The rest of the
lesson (contract types, backend JWT/login routes, route guards, frontend
login) is built incrementally in prompts B through F, each one runnable and
testable on its own.

## Running the app (Windows Terminal / PowerShell)

Prerequisites: Node.js installed, and a local SQL Server instance reachable
with the connection string in `apps/backend/.env` (see
`apps/backend/.env.example` for the supported formats — SQL auth, Windows
integrated auth, or a named instance).

From the **repository root**:

```powershell
# 1. Install dependencies for every workspace (root, apps/*, packages/*)
npm install

# 2. Apply Prisma migrations to your SQL Server database
npm run db:migrate --workspace=@equipment-hub/backend

# 3. Seed sample assets, technicians, work orders, and (from Lesson 2) users
npm run db:seed --workspace=@equipment-hub/backend

# 4. Start backend (Fastify, tsx watch) and frontend (Vite) together
npm run dev
```

Step 4 runs both apps concurrently in one terminal, prefixed `backend`/
`frontend` in blue/green. The frontend dev server prints the local URL to
open in your browser (Vite's default is `http://localhost:5173`); the
backend listens on the port configured in `apps/backend/.env`/`server.ts`.

Stop both with `Ctrl+C` in that terminal.

If you'd rather run them separately (e.g. to watch backend logs on their
own), open two terminals:

```powershell
# Terminal 1 — backend only
npm run dev --workspace=@equipment-hub/backend

# Terminal 2 — frontend only
npm run dev --workspace=@equipment-hub/frontend
```

Once auth is wired up (Prompts C–E), log in with one of the seed accounts
printed by the `db:seed` step (see the seed script's console output for the
current email/password pairs).

## What Prompt A asked for

> Add a `User` model to the Prisma schema (id, email, passwordHash, name,
> role, optional link to a `Technician`), generate a migration, and update
> the seed script to create sample users per role with bcrypt-hashed
> passwords.

## Steps that were actually followed

1. **Inspected the existing schema first.**
   Before changing anything, the current `apps/backend/prisma/schema.prisma`
   was read to see the existing `Asset`, `Technician`, and `WorkOrder`
   models and how they're mapped to SQL Server tables (`@@map`, `@db.*`
   attributes, `UniqueIdentifier` for GUID primary keys). This matters
   because the new model has to follow the same conventions instead of
   introducing a different style.

2. **Added the `User` model.**
   In `schema.prisma`, a new model was added:
   ```prisma
   model User {
     id           String   @id @default(uuid()) @db.UniqueIdentifier
     email        String   @unique
     passwordHash String
     name         String
     role         String   @db.VarChar(20)
     technicianId String?  @db.UniqueIdentifier
     createdAt    DateTime @default(now())

     technician Technician? @relation(fields: [technicianId], references: [id])

     @@map("users")
   }
   ```
   `role` is stored as a plain `VARCHAR(20)` (not a SQL Server enum — Prisma
   models enums as strings/constraints handled at the application layer,
   consistent with how `priority` and `state` are already stored on
   `WorkOrder`). `technicianId` is **nullable and optional**, because only
   `technician`-role users need to be linked back to a `Technician` row for
   "assigned tech" checks later; admins and supervisors don't have one.

   The inverse relation (`users User[]`) was added to the existing
   `Technician` model so Prisma can generate the relation both ways.

3. **Generated and applied a real migration.**
   ```bash
   npx prisma migrate dev --name add_users_and_roles
   ```
   This was run against the actual local SQL Server instance defined in
   `apps/backend/.env` (integrated auth, `localhost:1433`,
   `equipment_hub` database) — not a mocked or dry-run migration. Prisma
   generated `prisma/migrations/20260923063302_add_users_and_roles/migration.sql`,
   which creates the `users` table and its foreign key to `technicians`
   inside a transaction (`BEGIN TRAN` / `COMMIT TRAN` with rollback on
   error, matching the style of the existing `init` migration). It also
   regenerated the Prisma Client so `prisma.user.*` methods became
   available in code immediately.

4. **Added a password hashing dependency.**
   ```bash
   npm install bcryptjs
   npm install -D @types/bcryptjs
   ```
   `bcryptjs` (a pure-JS bcrypt implementation) was chosen over the native
   `bcrypt` package specifically to avoid native build/compilation issues
   on Windows, which is what this environment runs on. This is a practical
   trade-off worth noticing: pick the dependency that matches your actual
   dev environment, not just "whatever the tutorial used."

5. **Updated `prisma/seed.ts` to create sample users.**
   The seed script already loaded `assets.json`, `technicians.json`, and
   `work-orders.json` and inserted them via Prisma. It was extended to also:
   - Delete existing `users` rows first (`prisma.user.deleteMany()`), so the
     seed script stays idempotent like the rest of it.
   - Define four seed accounts as a plain array (`SEED_USERS`): one
     `admin`, one `supervisor`, and two `technician` accounts, each of the
     latter linked via `technicianId` to a **real, already-seeded**
     `Technician` id (Elena Vasquez and Marcus Chen) — this is what makes
     the "only the assigned technician can complete their own work order"
     rule testable later.
   - Hash each plaintext password with `bcrypt.hash(password, 10)` before
     inserting — passwords are never stored in plaintext, even in seed
     data.
   - Print the plaintext email/password pairs to the console at the end of
     the run, purely as a local development convenience so you can log in
     manually while testing — this is seed/dev-only behavior, never
     something you'd do in production code.

6. **Ran the seed script against SQL Server and read the output.**
   ```bash
   npm run db:seed
   ```
   Output confirmed 6 assets, 5 technicians, 15 work orders, and now 4
   users were created, followed by the printed login credentials table.
   This step matters pedagogically: a migration can succeed while a seed
   script still fails (e.g. a foreign key pointing at the wrong id), so
   both were verified independently against the real database rather than
   assumed to work from reading the code.

7. **Ran the full backend test suite twice.**
   ```bash
   npm test
   ```
   The first run showed 75/76 tests passing with one failure:
   `app.test.ts > assigns a known technician and only updates assignment
   fields` timed out at 5000ms. Re-running that file in isolation
   (`npx vitest run src/app.test.ts`) passed in 307ms, and a second full
   run passed all 76 tests. This was diagnosed as **pre-existing test
   flakiness unrelated to the schema change** (the test uses in-memory
   fixtures, not the new `User` table) rather than a regression — an
   important distinction: don't assume every red test was caused by your
   last change, but don't ignore it either. Confirm by isolating and
   re-running before concluding it's flaky.

## Why this order matters

Notice the sequence: **schema → migration → dependency → seed data →
run against the real database → run the test suite.** Each step was
verified before moving to the next, rather than writing all the code first
and hoping it works. This is deliberate — a migration that "looks right" in
a diff can still fail against a real SQL Server instance (wrong data type,
missing default, FK pointing at a non-existent row), so the only way to
know it's correct is to actually run it.

## What's next

- **Prompt B** — add `Role`, `User`, `LoginRequest`, `LoginResponse` types
  to the shared `packages/contract` package (OpenAPI spec + generated
  types), so both backend and frontend share one source of truth for these
  shapes.
- **Prompt C** — implement JWT issuing/verification and the
  `/api/auth/login` and `/api/auth/me` routes on the backend.
- **Prompt D** — gate the existing work-order routes (`cancel`, `complete`,
  `/assignment`) by role and technician ownership.
- **Prompt E** — add a login screen and auth-aware API client on the
  frontend.
- **Prompt F** — hide/show work-order action buttons in the UI based on the
  logged-in user's role and assignment.

Each of those prompts is scoped the same way Prompt A was: one layer of the
stack at a time, with its own tests, so you can verify correctness before
building the next layer on top.
