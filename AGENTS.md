# ToolLoop - Agent Reference

# What it is about 

Neighbourhood tool lending web app. We borrow tools from people nearby. No money, no real auth, no complexity.

# Stack

- Frontend : Next.js 16, App Router, TypeScript 
- SQLite + Prisma 7 + libsql adapter,
- Server Actions ONLY - no api routes, no client side fetchiing
- Plain css + css modules, no tailwind, no MUI, no component libraries
- Zod 4 validation , Biome 2 linting
- Demo auth via HTTP- only cookie (`toolloop_user`)

## Architecture
 UI -> Server Actions (Next.js)-> Domain (pure policy) -> Prisma

 - **Reads** : async server components query prisma directly

- **Writes** :Server actions -> Zod validation -> Domain check -> `$Transaction` -> `revalidatePath`-> return result


- **Domain Layer** (`src/lib/domain`): pure function , no input-output , no framework imports - all business logics rules lives here

- **Actions are thin** - orchestrate only, no rules, no raw SQL
 - **Components are dumb** : renders and dispatch only, never decide policy



 ## Setup
 
  ``` bash
  npm install
  npx prisma migrate dev
  npx prisma generate
  npx prisma db seed
  npm run dev
   
  ```


  Reset to clean state: `npm run db:reset`

  ## Key invariants 

  - Tool `available` and `BorrowRequest.status` must never drift- always mutate both in one `$Transaction`

  - A user cannot request they own tool or an unavailable tool
  - Approve request-> Approved -> + tool -> Unavailable (atomic)
 - Return : request -> Returned + tool -> Available (atomic), via otp only
 - Cancel: Borroweer cancels pending only , no otp needed , tool availability unchanged
 - Contact PII (email, phone, address) never rendered on public pages

 ## Data models
 - `User` -> owns `Tool`s, makes `BorrowRequest`s, has `Favourite`s
 - `Tool` -> has `BorrowRequest`s, has `Favorite`s, has `ToolReport`s
 - `BorrowRequest` -> status: `Pending -> Approved / Rejected -> Returned / Cancelled` has `RequestEvent`s
- `Favorite` -> `uniqu ([userId,toolId])` -> enforces toggle idempotency at DB level

- `RequestEvent` -> append-only audit log, written in same transaction as status change
-`ToolReport` -> private flag to tool owner, never public , never a score

## Folder conventions

- `src/lib/domain/` - pure business rules, no I/O  , no framework imports
- `src/actions/` - thin server actions
- `src/components/ui/`- styked primitives
- `src/lib/db.ts` - prisma client singletom (globalThis guard)
- `src/lib/sessions.ts`- `getCurrentUser()` , only place identity is resolved
- `src/lib/constants.ts`- ENUM -> label maps, never render raw enum strings

- CSS Modules co-located with each component , all values reference `:root` tokens in `global css`


## Routes

| ROUTE        | TYPE          | WHAT IT DOES                                                        |
|-------------|---------------|----------------------------------------------------------------------|
| `/`         | RSC           | Home page, hero, how it works, CTA                                   |
| `/browse`   | RSC           | Tool grid, filters via URL query params (`category`,`neighbourhood`,`available`,`q`) |
| `/tool/[id]`| RSC           | Tool detail, photo, owner info, rules, request button               |
| `/tools/new`| RSC + Actions | Create a new tool form, handle create tool action                   |
| `/dashboard`| RSC           | Owner's tools + incoming requests, approve/reject/return            |
| `/borrows`  | RSC           | Borrower's PENDING + APPROVED requests, CANCEL + OTP Return         |
| `/saved`    | RSC           | Borrower's saved tools, toggle favorite                             |


## Component and action conventions

**Server vs Client** all components are server components unless they need state or effects, then they are client components. All actions are server actions.

   

   ## What no to build
   - payments
   -rating/review
   -Image upload
   - real auth (email,pass , Oauth etc)
   - Real time chat
   - In-app ai
   - Multi -tenant (multiple neighbourhoods,organizations,etc)

   ## Prisma 7 gotchas
    - `Database_URL`  goes in `prisma.config.ts` , not in `schema.prisma`
    - Run `npx prisma generate` manually after every `migrate dev`
    - seed file must be `.mts` (ESM) to avoid CJS clash with libsql adapter 
    - Use `@defalul(uuid())` for UUID primary keys, cuid2 is not available 


    ## Every commit must

    - Pass `npm run lint` (Biome 2)
    - Pass `npm run build` with No type errors
-  Have a new caveman block appended to `checkpoint.txt` with a short description of the change 
- Be reachable via UI, not just in code
- Leave empty/loading/error states on every list surfaces (no blank)



