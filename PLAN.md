# Repwise: Spec & Implementation Plan

## Context
Repwise is a full-stack personal-training marketplace for a **1–2h YouTube tutorial**. The goal is to show the core flows and integrations (Neon Auth, Neon Realtime, Postgres, Stripe, Stream chat and video, ImageKit, Sentry, Cloudways) with a premium UI, without growing into a production platform.

The current repo (`repwise`) is a Next.js **16.4.3** / React 19.3 / Tailwind v4 starter that contains an unrelated habit tracker and a working light/dark toggle. We'll build here: delete the habit tracker and keep the theme toggle.

## Product
- **Clients** find trainers, buy workout programs (ordered video lessons), book paid 1:1 sessions, chat with trainers, join video calls, and leave reviews.
- **Trainers** manage a profile, programs and lessons, time slots, clients, and earnings in a dashboard.
- **Role is picked at onboarding** (Client or Trainer). A trainer is listed as soon as their profile is complete, with no approval step.

### In scope (v1)
- Trainer directory with a specialty filter (`?specialty=`) and featured trainers on the landing page
- Trainer profile showing bio, programs, open slots, reviews, and Message / Book buttons
- Programs: title, cover, price, and ordered lessons. **Lesson 1 is a free preview** and the rest unlock after purchase.
- Bookings: the trainer posts specific open slots (60 min) and sets one session price. The client picks a slot and pays with Stripe Checkout.
- 1:1 video call: one Stream call per booked slot (`callId = slotId`). Both sides get a "Join call" button, with no time window.
- Chat: any signed-in user can DM any trainer through Stream (replies, threads, reactions) on a `/messages` inbox.
- Reviews: 1–5 stars plus an optional comment. The average and count show on trainer cards and profiles.
- Trainer dashboard: earnings overview with a **live sales/bookings feed (Neon Realtime)**, program and lesson management with uploads, slots and bookings, and a clients list.
- Seed script with 4–6 trainers, programs, and slots for camera-ready data.

### Out of scope (v1, deferred)
Stripe Connect/payouts, subscriptions, refunds and cancellations, emails, admin panel, text search, lesson progress, weekly availability rules, group chats, unread badges/notifications, account deletion, multi-currency, i18n.

## Core flows
1. **Visitor** → landing page (hero, featured trainers, specialties) → `/trainers` → trainer profile → program page, where they can watch the preview lesson.
2. **Sign up** (Neon Auth prebuilt UI with email/password or Google) → `/onboarding`, which asks for a role. A trainer also fills in headline, bio, specialties, avatar, and session price.
3. **Buy a program**: the server action creates a Stripe Checkout session → payment → the webhook inserts a `purchase` → lessons unlock.
4. **Book a session**: the server action atomically **holds** the slot until Checkout expires (30 min) → Checkout → the webhook marks it `booked` and inserts a `purchase`. An abandoned hold simply expires.
5. **Session time**: both sides open `/call/[slotId]`. The server checks that the user is the slot's client or trainer, then joins the Stream call.
6. **Returning client** → `/library` shows their programs and upcoming sessions.
7. **Returning trainer** → `/dashboard`, where new sales and bookings show up live.

## Pages & routing
| Access | Routes |
|---|---|
| Public | `/`, `/trainers`, `/trainers/[id]`, `/programs/[id]` (`?lesson=` picks the lesson), Neon Auth pages |
| Signed in | `/onboarding`, `/library`, `/messages`, `/call/[slotId]` |
| Trainer only | `/dashboard`, `/dashboard/programs`, `/dashboard/programs/[id]`, `/dashboard/slots`, `/dashboard/clients` |
| API | `/api/webhooks/stripe` (the only route handler; everything else uses server actions) |

`proxy.ts` (Next 16) uses Neon Auth middleware to gate signed-in routes. The trainer role is checked in the dashboard layout, and every server action re-checks auth and ownership.

## Data model (Drizzle on Neon; auth users live in Neon Auth's schema)
- **profiles**: `userId` PK (the Neon Auth user ID), `role` (client|trainer), `name`, `avatarUrl`, plus nullable trainer fields: `headline`, `bio`, `specialties text[]`, `sessionPriceCents`.
- **programs**: `id`, `trainerId`→profiles, `title`, `description`, `coverUrl`, `priceCents`, `published`, `createdAt`.
- **lessons**: `id`, `programId`→programs (cascade), `title`, `position`, `videoPath`. `unique(programId, position)`. Position 1 is the preview.
- **slots** (the slot is also the booking): `id`, `trainerId`, `startsAt timestamptz`, `status` (open|held|booked), `clientId` (nullable), `heldUntil`.
- **purchases**: `id`, `clientId`, `trainerId`, `kind` (program|session), `programId?`, `slotId?`, `amountCents`, `stripeSessionId` **unique** (makes the webhook idempotent), `createdAt`. A partial unique index on `(clientId, programId)` prevents double-buying a program.
- **reviews**: `id`, `trainerId`, `clientId`, `rating` with CHECK 1–5, `comment?`, `createdAt`. `unique(trainerId, clientId)`, so a second submission edits the existing review. Only clients with a purchase from that trainer can review.
- Deletion: a program with buyers can only be unpublished, never deleted. Lessons can be deleted. A slot can be deleted only while it's open.
- Earnings are `SUM(amountCents)` per trainer (gross, no platform fee).

## Architecture
- **Reads**: Server Components query Drizzle directly. **Writes**: Server Actions, followed by `revalidatePath`. There's no client data-fetching library.
- **URL state**: the specialty filter and the selected lesson. Client state is limited to UI.
- **Realtime**: Neon Realtime (`@neon/realtime`) sealed live queries on `/dashboard` for recent purchases. The server seals the query and the client subscribes.
- **Money path**: Stripe Checkout (payment mode, USD, prices stored in cents) and one webhook for `checkout.session.completed`, with metadata carrying `kind`, `programId`, `slotId`, and `clientId`.

## Integrations: what each one owns
| Service | Owns |
|---|---|
| Neon Postgres | All app data. A `dev` branch for local work and `main` for prod. |
| Neon Auth (Managed Better Auth, beta) | Users and sessions. Email/password and Google. Prebuilt UI themed to our tokens. |
| Neon Realtime (beta) | Live sales and bookings feed on the trainer dashboard only |
| Stripe | Checkout and payments (test mode for the tutorial). Stripe sends receipts. |
| Stream | Chat (`stream-chat-react`: inbox, threads, reactions) and video (`@stream-io/video-react-sdk`). The server (`@stream-io/node-sdk`) upserts users and mints tokens. |
| ImageKit | Avatar and cover images (public, optimized). Lesson videos are uploaded from the browser as **private files** and played in the **ImageKit Video Player** through a `signerFn` server action that checks access before returning a signed URL valid for about 1h. |
| Sentry | Errors (client, server, edge), tracing, source maps, Session Replay, structured logs for checkout, webhook, and booking events, and Seer AI debugging (GitHub-connected) |
| Cloudways Velocity | Persistent Node server (`next build` / `next start`, Node 22/24 LTS) with env vars in the panel. Includes WAF, DDoS protection, and CDN. |

## UI
shadcn/ui + Tailwind v4 with a **dark-first athletic** look: near-black surfaces, one bold accent (electric lime), condensed display headings, and large trainer imagery. Light theme stays available through the existing toggle. Layouts are responsive and mobile-first for client pages; the dashboard is desktop-first but usable on mobile. Every list has a skeleton loading state and an empty state.

## ASSUMPTIONS (no firm answer given)
1. We build in this repo; the habit tracker is deleted and the theme toggle is kept.
2. USD only, prices stored as integer cents, sessions fixed at 60 min.
3. Times are stored as `timestamptz` and shown in the viewer's local time zone.
4. Earnings are gross, with no platform fee.
5. Review eligibility is any paid purchase from that trainer; one review per client per trainer, editable.
6. Seeded trainers are created as real Neon Auth users with a demo password, so you can log in as one to demo a full dashboard.
7. Only `local` (Neon `dev` branch) and `production` (Neon `main`) environments exist, with Stripe test mode in both. There's no staging or CI; Velocity deploys from git.
8. SEO: `generateMetadata` and OG on the landing, trainer, and program pages; `robots.ts` disallows app routes. No sitemap.
9. Video uploads are capped at 100 MB on the client, pending ImageKit plan limits.
10. "Done" means a manual end-to-end demo script passes on production: sign up → buy → watch → book → chat → call → review → trainer sees the sale live. Plus one small runnable check on the access and slot-hold logic.
11. No rate limiting beyond the Cloudways WAF. Stripe and Stream handle abuse on their own surfaces.

## OPEN RISKS
1. **Three beta dependencies**: Neon Auth, Neon Realtime, and the ImageKit Video Player SDK. Their APIs could change mid-recording, so we'll pin versions and re-check the docs at implementation time.
2. **Sentry Seer needs a paid plan or trial** plus a GitHub connection, and its pricing changed in January 2026. Confirm your plan before recording that segment.
3. **ImageKit private files + adaptive streaming (HLS) + `signerFn`**: I couldn't find a documented example of the three together. The fallback is signed progressive MP4 in the same player.
4. **Stream call permissions**: the default call type's join rules need checking so only the two members can join; the server-side page check is in place either way.
5. **Slot holds**: Stripe Checkout's minimum expiry is 30 min, so an abandoned checkout blocks a slot for up to 30 min. This is acceptable for v1.
6. **Next 16 conventions** (`proxy.ts`, caching APIs): per AGENTS.md, read `node_modules/next/dist/docs/` before writing code.
7. **Velocity build environment**: `NEXT_PUBLIC_*` values and `SENTRY_AUTH_TOKEN` must be available at build time, not only at runtime.
8. **Free-tier limits** for ImageKit bandwidth/storage and the Stream plan during recording and after the video goes live.

---

# Implementation Plan

Each phase below is one tutorial chapter and ends on something you can demo. Per AGENTS.md, re-read the matching guide in `node_modules/next/dist/docs/01-app/` before each phase. Re-check the docs for the vendor packages marked *(verify)*, since their APIs are new or in beta.

## Next 16 ground rules (from bundled docs)
- **`cacheComponents: true` is already on** in [next.config.ts](next.config.ts). Every per-user or uncached DB read lives in an async component wrapped in `<Suspense>` with a skeleton fallback, which also gives us the loading states (`01-getting-started/08-caching.md`).
- No `'use cache'` in v1. `// ponytail: all reads dynamic; add 'use cache' + cacheTag on the public directory if TTFB matters.`
- Mutations are `<form action={serverAction}>`. Input is validated with `zod` on the server, then `revalidatePath()` (`07-mutating-data.md`). No react-hook-form; native `required`/`min`/`type` attributes handle client-side hints.
- `proxy.ts` (not `middleware.ts`) is an optimistic gate only. Real auth checks happen in layouts and server actions (`16-proxy.md`, `02-guides/authentication.md`).

## File layout (keep it flat)
```
proxy.ts                      Neon Auth gate for signed-in routes
drizzle.config.ts
lib/db/schema.ts              6 tables + enums + constraints
lib/db/index.ts               drizzle(neon(DATABASE_URL))
lib/auth.ts                   createNeonAuth (server) + getProfile/requireUser/requireTrainer
lib/auth-client.ts            createAuthClient (client)
lib/access.ts                 pure rules: canWatchLesson, canReview (+ access.test.ts)
lib/stripe.ts, lib/stream.ts, lib/imagekit.ts, lib/realtime.ts   one SDK client each
app/actions/*.ts              profile, programs, slots, checkout, reviews, media, stream
app/api/webhooks/stripe/route.ts
app/(site)/…                  landing, trainers, programs, library, messages, call, onboarding
app/dashboard/…               overview, programs, programs/[id], slots, clients
scripts/seed.ts
instrumentation*.ts, sentry configs (generated by the wizard)
```

## Phase 0: Cleanup & design foundation
- Delete the habit tracker from [app/page.tsx](app/page.tsx). Keep the theme toggle's inline head script in [app/layout.tsx](app/layout.tsx) and the `data-theme` dark variant in [app/globals.css](app/globals.css), but **default to dark**.
- `npx shadcn@latest init`, then add button, card, badge, dialog, tabs, table, input, textarea, select, sonner, skeleton, avatar, and dropdown-menu.
- Tokens in `globals.css`: near-black surfaces and an electric-lime accent. Fonts via `next/font`: a condensed display face plus Inter.
- App shell: a navbar with logo, Trainers, Library/Dashboard by role, Messages, the theme toggle, and the user button. Build the landing page with static sections first.
- ✅ The dark, premium landing page renders.

## Phase 1: Database (Neon + Drizzle)
- Create the Neon project and a `dev` branch. Install `drizzle-orm @neondatabase/serverless` and `-D drizzle-kit`.
- `lib/db/schema.ts` covers profiles, programs, lessons, slots, purchases, and reviews, with enums, `unique(stripeSessionId)`, a partial unique `(clientId, programId) WHERE kind='program'`, `unique(programId, position)`, `unique(trainerId, clientId)` on reviews, and a `CHECK rating BETWEEN 1 AND 5`.
- `npx drizzle-kit push` for dev; `generate` + `migrate` for prod.
- ✅ The tables are visible in the Neon console.

## Phase 2: Auth + onboarding (Neon Auth)
- Enable Neon Auth with email/password and Google. Install `@neondatabase/auth` *(verify)*. Mount the prebuilt UI auth pages and theme them with our tokens.
- `proxy.ts` uses `auth.middleware({ loginUrl })` and matches `/onboarding`, `/library`, `/messages`, `/call`, and `/dashboard`.
- `lib/auth.ts` provides `getProfile()`. Signed-in users without a profile are redirected to `/onboarding`. `requireTrainer()` is used in `app/dashboard/layout.tsx`.
- `/onboarding` offers role cards (Client/Trainer). The trainer form takes headline, bio, specialty chips, and session price. A server action inserts the profile.
- ✅ Sign up with Google → pick Trainer → land on `/dashboard`. Sign up as a client → land on `/trainers`.

## Phase 3: ImageKit uploads
- Install `@imagekit/next` *(verify)*. The `getUploadAuth()` server action requires a signed-in user and returns the token, signature, and expiry.
- A reusable `<Uploader>` client component calls `upload()` with a progress bar and accepts `isPrivate`, `folder`, and `maxBytes` (100 MB for video).
- Images are public and use the ImageKit `<Image>` with transformations. Add avatar upload to onboarding.
- ✅ An avatar uploads and shows an optimized result in the navbar.

## Phase 4: Programs, lessons & the video player
- Dashboard `programs` lists programs. `programs/[id]` is the editor: title, description, price, cover upload, publish toggle, and a lesson list. "Add lesson" uploads a **private** video to `/lessons/{programId}`. Lessons are ordered by `position`, and the lowest one is the free preview. Deleting a lesson is allowed; a program with purchases can only be unpublished.
- Public pages:
  - `/trainers` is a grid with specialty chips (`?specialty=`) and the average rating.
  - `/trainers/[id]` shows the profile, programs, open slots, and reviews.
  - `/programs/[id]` shows the lesson list, the player, and `?lesson=`.
- `lib/access.ts` defines `canWatchLesson({ isPreview, isOwner, hasPurchase })`. The `signLessonUrl(lessonId)` server action loads the lesson, checks access, and returns an ImageKit signed URL that expires in 1h.
- Player: the ImageKit Video Player React component *(verify package)* with `signerFn` → `signLessonUrl`, and `abs: { protocol: 'hls' }`. **Fallback** if signed private files and HLS don't combine: drop `abs` and play the signed progressive MP4.
- ✅ The preview lesson plays when logged out. Locked lessons show a lock and the "Buy" CTA, and a copied signed URL dies after it expires.

## Phase 5: Seed data
- `scripts/seed.ts`, run with `npx tsx`, creates 5 trainers as real Neon Auth users (via the auth sign-up API *(verify)*, password `demo1234`). It adds profiles, 1–2 programs each with 3 lessons, and future open slots. Images and one sample video are uploaded once to ImageKit `/seed` and reused.
- ✅ `/trainers` looks full, and you can log in as a seeded trainer.

## Phase 6: Stripe — program purchase
- `lib/stripe.ts`. The `buyProgram(programId)` action requires a client who hasn't already bought it, then creates a Checkout session (payment mode, `price_data`, metadata `{kind, programId, clientId, trainerId}`, success URL back to the program) and redirects.
- `app/api/webhooks/stripe/route.ts` reads `await req.text()` and calls `constructEvent` with the secret. On `checkout.session.completed` it inserts the purchase with **`onConflictDoNothing` on `stripeSessionId`** (idempotent), then `revalidatePath`.
- Local testing: `stripe listen --forward-to localhost:3000/api/webhooks/stripe`.
- ✅ Buy with `4242…` → lessons unlock. `stripe events resend <id>` creates no duplicate row.

## Phase 7: Slots & bookings
- Dashboard `slots` has a native `<input type="datetime-local">` to add a slot, a list of upcoming slots with status, and delete while open.
- `bookSlot(slotId)` runs one atomic SQL hold: `UPDATE slots SET status='held', client_id=$me, held_until=now()+'30 min' WHERE id=$id AND trainer_id<>$me AND starts_at>now() AND (status='open' OR (status='held' AND held_until<now())) RETURNING *`. If no row comes back, show "Slot just got taken". Otherwise create a Checkout session with `expires_at = held_until` and redirect.
- In the webhook, `kind='session'` inserts the purchase and runs `UPDATE slots SET status='booked' WHERE id AND client_id = metadata.clientId`. If 0 rows are updated, log `Sentry.captureMessage` because it needs a manual refund.
- Open slots query: `status='open' OR (status='held' AND held_until<now())`, future only. Times are formatted in the viewer's time zone by a small client component.
- `/library` shows the client's purchased programs and booked sessions with a "Join call" button.
- ✅ Two browsers race for one slot and only one reaches Checkout. A paid slot shows up in both libraries.

## Phase 8: Stream chat
- `lib/stream.ts` uses the `@stream-io/node-sdk` server client. The `getStreamToken()` action upserts the user (id, name, image) and returns a token.
- On the trainer profile, the "Message" action creates a distinct `messaging` channel with members `[me, trainerId]` and redirects to `/messages?c=<cid>`.
- `/messages` is a client component using `stream-chat-react`: `Chat`, `ChannelList` (filter `members $in [me]`), `Channel`, `Window`, `MessageList`, `MessageInput`, and `Thread`. Replies, threads, and reactions are built in. Theme it with Stream's CSS variables in dark mode.
- ✅ A client DMs a trainer, the trainer replies in a thread, and a reaction appears on both sides.

## Phase 9: Stream video calls
- The `/call/[slotId]` server page loads the slot, calls `notFound()` unless the user is its client or trainer and it's booked, then renders the client `<CallRoom>`.
- `<CallRoom>` uses `@stream-io/video-react-sdk`: `StreamVideoClient` (token from `getStreamToken`), `call('default', slotId).join({ create: true, data: { members } })`, then `SpeakerLayout` and `CallControls`. Leaving goes back to the library or dashboard.
- Check the default call type's permissions in the Stream dashboard so non-members can't join *(open risk 4)*.
- ✅ The client and trainer are in a 1:1 call from two browsers, and a third account gets a 404.

## Phase 10: Reviews
- `canReview` means the client has a purchase from this trainer. A form on the trainer profile (star radio buttons + textarea) calls a `submitReview` action that upserts `onConflictDoUpdate (trainerId, clientId)`.
- The directory and profile query `AVG(rating)` and `COUNT(*)`.
- ✅ A purchaser leaves 5★, and the card shows ★5.0 (1). A non-purchaser doesn't see the form.

## Phase 11: Trainer dashboard + Neon Realtime
- Overview shows total earnings, number of sales, and upcoming sessions. **The live feed** uses `lib/realtime.ts`: `createRealtime` from `@neon/realtime/server` *(verify)*. The server seals `SELECT … FROM purchases WHERE trainer_id=$me ORDER BY created_at DESC LIMIT 10` plus a `SUM(amount_cents)` query, and a client component subscribes to them.
- The `clients` page shows distinct clients from purchases joined with profiles: what they bought, their last purchase, and a Message button.
- ✅ With the trainer dashboard open in browser A and a purchase made in browser B, the sale and total update without a refresh.

## Phase 12: Sentry
- Run `npx @sentry/wizard@latest -i nextjs`. Enable tracing, Session Replay, and `enableLogs`. Add `Sentry.logger.info` calls at checkout created, webhook processed, slot held, and slot booked, and `captureException` in the webhook's catch.
- Connect the GitHub repo in Sentry and enable Seer (needs a paid plan or trial — open risk 2).
- Demo: ship a deliberate bug (for example, an unguarded `trainer.specialties.map` on a profile with null specialties), trigger it, then watch the replay and run Seer's root cause and fix.
- ✅ The error appears with a readable stack, a replay, and a trace, and Seer proposes the fix.

## Phase 13: Deploy to Cloudways Velocity
- Run `drizzle-kit migrate` against Neon `main`, then run the seed against prod.
- Velocity app from the GitHub repo on Node 22/24, with `npm run build` / `npm start`. Set **all env vars, including `NEXT_PUBLIC_*` and `SENTRY_AUTH_TOKEN`, at build time**.
- Production wiring:
  - Stripe webhook endpoint `https://<domain>/api/webhooks/stripe`, with the new `STRIPE_WEBHOOK_SECRET`
  - Neon Auth trusted domain
  - Google OAuth redirect URI
  - ImageKit and Stream keep the same keys
- ✅ The full demo script below passes on the production URL.

## Env vars
`DATABASE_URL`, `NEON_AUTH_BASE_URL`, `NEON_AUTH_COOKIE_SECRET`, `NEON_REALTIME_URL`, `NEON_REALTIME_SECRET`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `NEXT_PUBLIC_APP_URL`, `NEXT_PUBLIC_STREAM_API_KEY`, `STREAM_API_SECRET`, `NEXT_PUBLIC_IMAGEKIT_URL_ENDPOINT`, `NEXT_PUBLIC_IMAGEKIT_PUBLIC_KEY`, `IMAGEKIT_PRIVATE_KEY`, `NEXT_PUBLIC_SENTRY_DSN`, `SENTRY_AUTH_TOKEN`, `SENTRY_ORG`, `SENTRY_PROJECT`. Secrets are never `NEXT_PUBLIC_`.

## Verification
- **One runnable check:** `node --test lib/access.test.ts` (Node 22+/24 strips TS types) asserts the `canWatchLesson`/`canReview` rules: preview open, owner open, purchaser open, everyone else locked.
- **Money path:** after `stripe events resend`, there's still exactly one purchase row. The two-browser slot race leaves exactly one holder.
- **`npm run build`** passes with no blocking-route warnings, meaning every dynamic read is inside `<Suspense>`.
- **End-to-end demo script (definition of done):** a visitor watches the preview → signs up as a client → buys a program → watches lesson 2 → books a slot → messages the trainer (reply, thread, reaction) → both join the call → the client leaves a review → the trainer dashboard shows the sale live and the client in the clients list → a forced error appears in Sentry with a replay and Seer analysis.
