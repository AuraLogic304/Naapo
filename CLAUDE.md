# CLAUDE.md

Project instructions for AI agents working in this repository. Read this fully before writing code.

---

## 1. What we are building

A B2B mobile-first platform for **Indian apparel retail shops**. It is **not** a POS and **not** an e-commerce site.

Four surfaces, one product:

| Surface | Who | What it does |
|---|---|---|
| **Staff app** | Shop floor staff | Take customer measurements, run a virtual fitting, generate alteration tickets |
| **Owner view** | Shop owner | Daily numbers, size-demand intelligence, staff performance |
| **Customer page** | Walk-in customer, via QR | See their avatar + fit, share to WhatsApp family |
| **Admin web** | Owner / back office | Catalogue, garment measurements, staff accounts, settings |

**The core loop:**

```
Customer walks in → staff measures with a tape → fit profile created
   → fitting shows what fits → stock check (here / other branch / order)
   → sale, OR alteration ticket to the tailor
   → profile saved → WhatsApp follow-up later
```

**The three things that make this product work.** If a change weakens any of these, stop and flag it:

1. **Second visit is 15 seconds.** First fitting takes ~3 minutes. Every visit after: phone number in, straight to results.
2. **"Doesn't fit" is never the answer.** In India, alteration is cheap and expected. The output is *"size 40, take in waist 4cm, ₹150, ready Thursday"* — a sale, not a rejection.
3. **The owner's real interface is WhatsApp.** A 9pm nightly summary. He may never open the app; he must still get value daily.

---

## 2. Stack

**Decided. Do not substitute without asking.**

| Layer | Choice |
|---|---|
| Mobile | **Expo (React Native) + TypeScript** |
| Web | **React + Vite + TypeScript** |
| Backend | **Supabase** — Postgres, Auth, Storage, Edge Functions, RLS |
| Local DB / sync | **PowerSync** over expo-sqlite |
| Monorepo | **pnpm workspaces + Turborepo** |
| 3D avatar | **three.js via expo-gl** + react-three-fiber |
| Styling (mobile) | NativeWind |
| Styling (web) | Tailwind + shadcn/ui |
| State | Zustand (client) + TanStack Query (server) |
| Forms/validation | Zod (shared between client and server) |
| Barcode | react-native-vision-camera |
| Voice | Android native SpeechRecognizer (hi-IN, gu-IN, en-IN) |
| WhatsApp | Indian BSP (AiSensy / Interakt / Wati) — **not** Meta Cloud API directly |
| Errors | Sentry |

**No** Unity, no Unreal, no GraphQL, no microservices, no Kubernetes, no Redux, no custom design system.

---

## 3. Repository layout

```
/apps
  /mobile          Expo app — staff AND owner (see §3.1)
  /web             React + Vite — admin/catalogue dashboard
  /customer        React + Vite — public QR landing page, tiny bundle
/packages
  /fit-engine      ★ Pure TS. Fit, grading, alteration rules. Zero I/O.
  /db              Supabase generated types, query helpers, sync schema
  /avatar          three.js avatar mesh, blendshape driver, glTF loading
  /i18n            Translation keys — en, hi, gu
  /shared          Zod schemas, constants, units, formatting
  /config          eslint, tsconfig, tailwind presets
/supabase
  /migrations      SQL migrations — the source of truth for schema
  /functions       Edge Functions (WhatsApp send, nightly summary, sync hooks)
```

### 3.1 One binary, two apps

Staff and owner ship as **one Expo binary** but have **completely separate navigation stacks**, chosen by role at login. An owner must never see a staff screen and vice versa. Do not build a shared screen with role-conditional rendering — branch at the root navigator.

### 3.2 packages/fit-engine is sacred

- **Pure functions only.** No network, no database, no React, no Supabase imports.
- Must run identically on device (offline) and in an Edge Function.
- Every rule needs a unit test with a real-world example.
- If you're tempted to `await` something in here, you're in the wrong package.

---

## 4. Domain vocabulary

Get these wrong and the whole product is wrong.

| Term | Meaning |
|---|---|
| **Body measurement** | Measured on the person. Chest, waist, hip, shoulder, sleeve length, etc. |
| **Garment measurement / PTM** | Point of measure. Measured on the *garment*, flat on a table. |
| **Flat measurement** | Half the circumference. A garment chest of 21" flat = 42" around. **Always double before comparing to a body girth.** |
| **Ease** | `garment girth − body girth`. The room in the garment. Drives the fit verdict. |
| **Base size** | The one size physically measured for a style. Others are derived. |
| **Grading** | Size-to-size increments (e.g. +2" chest per size). Used to derive all sizes from the base size. |
| **Take in / let out** | Reduce / increase a garment measurement. Letting out is limited by seam allowance. |
| **Shorten / lengthen** | Length alterations. Lengthening is limited by hem allowance. |
| **Alterability** | Per category and zone: can it be altered, in which direction, by how much, at what cost. |
| **Fit verdict** | `GREEN` fits as-is · `YELLOW` fits after alteration · `RED` won't work |

---

## 5. Hard conventions

### 5.1 Units — measurements

**Store every measurement as an integer in millimetres.** Never floats. Never inches in the database.

```ts
type Millimetres = number & { readonly __brand: 'mm' };

// Display converts at the edge, per shop preference (inch or cm)
formatMeasurement(1016 as Millimetres, 'inch')  // "40""
formatMeasurement(1016 as Millimetres, 'cm')    // "101.6 cm"
```

Indian shops speak in **inches** for garments. Default display is inches; storage is always mm.

### 5.2 Units — money

**Store as integer paise.** Never floats, never rupees-as-decimal.

```ts
type Paise = number & { readonly __brand: 'paise' };
formatMoney(15000 as Paise)  // "₹150"
```

### 5.3 Every table

Non-negotiable columns on every domain table:

```sql
id           uuid primary key default gen_random_uuid()  -- client may generate
shop_id      uuid not null references shops(id)
created_at   timestamptz not null default now()
updated_at   timestamptz not null default now()
deleted_at   timestamptz                                  -- soft delete only
```

- **Client generates UUIDs.** Offline creation must work without a server round trip.
- **Soft delete only.** Sync cannot handle hard deletes.
- **RLS is mandatory on every table.** No exceptions. A shop must never see another shop's data.

### 5.4 Strings

No hardcoded user-facing English. Ever.

```ts
// ✗
<Text>Perfect fit</Text>
// ✓
<Text>{t('fit.verdict.perfect')}</Text>
```

Languages: `en`, `hi`, `gu`. Add keys to `packages/i18n` in all three, even if hi/gu are placeholders initially.

---

## 6. Offline-first

**A complete fitting must work with no network.** This is a hard requirement, not a nice-to-have. Shop internet fails constantly.

Rules:

1. **Never block the UI on a network call.** Write locally, sync in the background.
2. **Never show a network error to staff.** Show a small grey cloud icon in the header. Nothing else.
3. **Read from local SQLite, never from the network, on the fitting path.**
4. **Conflict resolution is last-write-wins on `updated_at`.** Our data is scoped to one shop with few concurrent writers — this is genuinely sufficient. Do not build CRDTs.
5. **The fit engine and avatar rendering run on-device.** Nothing on the fitting path may require a server.

Data that must be available offline: customer profiles, measurements, garment catalogue with PTMs, grade rules, alterability rules, stock snapshot (with a visible "last updated" timestamp), alteration tickets.

Data that may be online-only: analytics, owner reports, WhatsApp sending, cross-branch stock lookup (degrade gracefully — show last-known with the timestamp).

---

## 7. Auth model

Shops share devices. Do not require a phone OTP for every staff login.

```
Device registration (once)   → shop links the device, stores a long-lived token
Staff switch (every shift)   → tap avatar, enter 4-digit PIN
Owner login                  → phone + OTP
Customer QR page             → no auth, short-lived signed token in the URL
```

Staff accounts are created by the owner in the admin web app. Staff never self-register.

---

## 8. The fit engine

Located in `packages/fit-engine`. This is the heart of the product.

```ts
export function evaluateFit(input: {
  body: BodyMeasurements;          // mm
  garment: GarmentMeasurements;    // mm, flat
  category: Category;
  fabric: FabricProperties;        // stretch %, construction
  alterability: AlterabilityRules;
}): FitResult;

export type FitResult = {
  verdict: 'GREEN' | 'YELLOW' | 'RED';
  zones: ZoneFit[];                       // per zone: ease, classification
  alterations: Alteration[];              // what to change, by how much
  alterationCostPaise: Paise;
  alterationDays: number;
  blockedReason?: string;                 // only on RED
  confidence: 'HIGH' | 'MEDIUM' | 'LOW';  // LOW when garment data is estimated
};
```

Order of evaluation:

1. Compute ease per zone (double flat measurements first).
2. Add stretch allowance: `bodyGirth × stretchPct × 0.35`. People don't wear fabric at max stretch.
3. Classify each zone against `CATEGORY_EASE_THRESHOLDS`.
4. If any zone is out of range, check alterability — direction, maximum, seam allowance.
5. Assign verdict. Compute cost and turnaround from the shop's alteration rate card.

**Threshold tables live in `packages/fit-engine/rules/`** as plain data, per category. These are proprietary IP and will be tuned from real feedback — keep them as data, never inline in logic.

**On RED, always return alternatives.** A red screen is never a dead end. Query order: same style different size in this shop → similar style that fits → same style at another branch.

**Confidence must be `LOW` when garment measurements are graded rather than measured, or inferred from a size chart.** The UI must show this. Never present an estimate as certain.

---

## 9. UI rules

These come from the product spec and are not negotiable. If a design breaks one, flag it before implementing.

1. **One job per screen.** One primary action. Never two competing CTAs.
2. **Three taps maximum** from home to any core action.
3. **Icon first, text second.** Every button has a picture.
4. **Voice input on every measurement field.** Staff have both hands on the tape.
5. **Minimum touch target 56px. Minimum body text 18px. Result numbers 40px+.**
6. **Never a blank state.** Every empty screen has an explanation and a button.
7. **Never a dead end.** Every "no" carries alternatives.
8. **Staff screens and owner screens are different products.** No shared dashboards.

Design tokens live in `packages/config/tokens.ts`. Use them; don't hardcode sizes or colours.

**Fit verdict colours are fixed:**
`GREEN #16A34A` · `YELLOW #EAB308` · `RED #DC2626`

---

## 10. Avatar

- Single low-poly rigged mesh, **12–16 blendshapes** driven directly by measurements.
- glTF + Draco compression. **Under 1.5MB total.**
- Stylised, matte, **faceless**. No PBR skin, no hair physics, no photorealism.
- Renders on-device via `expo-gl`. Must work offline.
- Skin tone: 6 options, applied as a material colour.
- Fit zones overlay as vertex colours — green/yellow/red on the mesh.

Do not add photorealistic rendering, cloth simulation, or server-side rendering. Those are explicitly out of scope for v1.

---

## 11. WhatsApp

All WhatsApp traffic goes through a BSP, called from a Supabase Edge Function. Never from the client.

Message types (all need Meta template pre-approval — plan ahead, this blocks launch):

| Template | Trigger |
|---|---|
| `owner_daily_summary` | pg_cron, 21:00 shop-local |
| `alteration_ready` | Tailor scans the ticket QR |
| `alteration_overdue` | Cron, when past due |
| `share_look` | Staff or customer taps "Send to family" |
| `size_restock` | Owner broadcasts to size-matched customers |

Never send unsolicited messages. Consent is captured at profile creation and stored on the customer record.

---

## 12. Never do these

- **Never store customer photos.** Process in memory, extract what's needed, discard. Nothing written to Storage.
- **Never build face recognition** or anything that identifies a person by body shape. It changes our legal position under the DPDP Act.
- **Never use floats** for money or measurements.
- **Never hard delete.** Soft delete only.
- **Never block the fitting flow on the network.**
- **Never commit to `main`.** Branch → PR → review → merge.
- **Never add a table without RLS.**
- **Never hardcode a user-facing string.**
- **Never present an estimated garment measurement as confident.**
- **Never build billing, GST, or invoicing.** We integrate; we don't compete with Vyapar/Marg/Busy.
- **Never add a screen with two primary actions.**

---

## 13. Commands

```bash
pnpm install

pnpm dev:mobile          # Expo dev server
pnpm dev:web             # admin dashboard
pnpm dev:customer        # customer QR page

pnpm test                # all packages
pnpm test:fit            # fit-engine only — run this on any rules change

pnpm db:migrate          # apply Supabase migrations
pnpm db:types            # regenerate TS types from schema — after every migration
pnpm db:reset            # local reset + seed

pnpm lint && pnpm typecheck
```

**After any schema change: migration first, then `pnpm db:types`, then update `packages/db`.** Never hand-edit generated types.

---

## 14. Testing

- **`fit-engine`: unit tests are mandatory.** Every rule, every category, with real measurements. This is the one place bugs are expensive — a wrong recommendation destroys staff trust permanently.
- **Sync: test the offline path.** Airplane mode → complete a fitting → reconnect → verify.
- **Test on a real mid-range Android** (₹12–18k, 4GB RAM). Not just a simulator. This is what staff actually use.
- UI tests: only for the measurement flow and the fit result screen. Don't chase coverage elsewhere.

---

## 15. Working style

- Small PRs. One concern each.
- If a task is ambiguous, ask before building. Do not guess at product behaviour.
- If a change would break one of the three core things in §1, say so before implementing.
- Prefer boring, obvious code. Three people maintain this; cleverness costs more than it saves.
- When adding a dependency, justify it in the PR description. Bundle size matters — staff phones are cheap and shop internet is slow.
