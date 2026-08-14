# Chef KnifeWorks — website + Shop OS

A storefront that takes bookings and a back office that runs the bench, the
customers, the money, and every message the shop sends. React + Vite + TypeScript
+ Tailwind on the front, Supabase (Postgres) on the back.

The rules live in the database, not in the app. Pricing, capacity, referral
payouts, invoice totals, and consent are all enforced by constraints, triggers,
and row level security — so a phone, a laptop, or a future integration all
behave the same way and there is no path around them.

---

## Get it running

```bash
npm install
cp .env.example .env          # fill in your Supabase URL and anon key

npm run db:verify             # runs every migration + 36 behaviour checks, offline
npm run dev                   # http://localhost:5173
```

`db:verify` boots a real Postgres in WebAssembly, applies the migrations, and
drives one full day of business through them: book a job, take in the knives,
sharpen them, invoice, take partial then full payment, and pay out a referral.
It needs no network and no Supabase project. **Run it before every deploy.**

### Point it at a Supabase project

```bash
npx supabase init
npx supabase link --project-ref YOUR-REF
npm run db:push               # applies supabase/migrations in order
npm run db:types              # regenerates src/lib/database.types.ts (optional)
```

Then create your own staff login:

```sql
-- after signing up through Supabase Auth
insert into profiles (id, full_name, email, role)
values ('<auth user id>', 'Jason', 'you@chefknifeworks.com', 'owner');
```

### Turn on messaging

```bash
npx supabase secrets set PLIVO_AUTH_ID=... PLIVO_AUTH_TOKEN=... \
  PLIVO_FROM_NUMBER=+1612XXXXXXX RESEND_API_KEY=... \
  RESEND_FROM="Chef KnifeWorks <hello@chefknifeworks.com>" \
  GOOGLE_REVIEW_URL=... PUBLIC_SITE_URL=https://chefknifeworks.com
npx supabase functions deploy dispatch-messages
```

Then schedule it once a minute with `pg_cron` + `pg_net` — the SQL is in the
comment at the top of `supabase/functions/dispatch-messages/index.ts`. Until it
is scheduled, messages pile up in the queue and are visible under **Messages**;
nothing is lost.

---

## How the shop moves through it

```
website /book ──▶ request_booking()  one RPC: finds or creates the customer,
                                     honours a referral code, prices the cart
                                     from the live catalog, holds the slot
       │
       ▼
Schedule ──▶ Confirm ──▶ Take in ──▶ open_work_order()  one line per blade
       │
       ▼
Bench board: intake · triage · 220 · 400 · 1000 · 3000 · strop · QC · ready
       │                                        (customer watches the same rail)
       ▼
issue_invoice()  built from what was actually done — angle and finishing stone
       │         land on the invoice line, store credit applies automatically
       ▼
Payment ──▶ receipt · CRM rollups · referral payout · review request in 48h
```

### Where things are

| Path | What it holds |
| --- | --- |
| `supabase/migrations/0001` | Extensions, enums, staff, customers, knives, catalog |
| `supabase/migrations/0002` | Availability windows, bookings, work orders, bench history |
| `supabase/migrations/0003` | Invoices, payments, store-credit ledger, referrals, memberships |
| `supabase/migrations/0004` | Templates, automations, timeline, message outbox |
| `supabase/migrations/0005` | Triggers, RPCs, reporting views — the business logic |
| `supabase/migrations/0006` | Row level security |
| `supabase/migrations/0007` | Seed: services, hours, tiers, message copy |
| `src/site/` | Public site: home, booking, tracking, referral landing |
| `src/os/` | Shop OS: today, bench, schedule, customers, invoices, referrals, messages, settings |
| `src/data/api.ts` | Every read and write. Components never call Supabase directly. |
| `src/lib/bench.ts` | The protocol — stage order, grits, channel labels |

---

## Decisions worth knowing about

**Capacity is bench-minutes, not appointments.** A restaurant route with forty
blades and one home cook are not the same job. Each service carries
`bench_minutes`; `available_slots()` subtracts what is already booked from each
window's capacity and only returns slots that still fit your cart.

**Invoices are built from the bench, never from the booking.** The quote at
booking is a quote. `issue_invoice()` reads the work order items — including the
angle and finishing grit you typed while sharpening — so the customer's invoice
explains what they paid for. Both `open_work_order()` and `issue_invoice()` are
idempotent: click twice, get the same record.

**Referrals pay out on money, not on signups.** A referral qualifies when the
new customer's *first invoice is actually paid*. Self-referral is blocked by
constraint; a customer can only ever be referred once.

**Store credit is a ledger.** `customers.credit_cents` is a cached rollup kept
by trigger. Every dollar is traceable to a reason, and the balance can never go
negative.

**Consent is enforced in the database.** `emit_journey()` will not queue an SMS
to anyone without `sms_opt_in`, so no future feature can accidentally text
someone who said no. Every queued message carries a dedupe key — the same
notification can never go out twice.

**The public gets three functions and two tables.** Anonymous visitors can call
`available_slots`, `request_booking`, and `track_order`, and read active services
and locations. That is the entire surface. Prices are looked up server-side, so
a tampered browser payload cannot change what a job costs. The tracking link
returns progress and a first name — no phone, no email, no totals.

**Views run with `security_invoker`.** Without it a Postgres view executes as its
owner and quietly bypasses row level security on the tables underneath.

---

## Design

The palette comes off the bench: carbon steel, the grey-green slurry a
waterstone throws, brass rivets, and the paper of a laminated spec card. The
storefront is light, the Shop OS is dark — one is a shop window, the other is a
work surface under task lighting.

The signature element is the **grit rail**. A knife's progress through the shop
*is* a grit progression, coarse to fine, so the real sequence is the progress
indicator — on the marketing page, on the bench card, and on the customer's
tracking screen. The numbers under each step are the actual stones. Numbered
markers earn their place here because the content genuinely is a sequence.

Type is Bricolage Grotesque for display, Public Sans for reading, and JetBrains
Mono for anything that is craft data: grits, angles, references, money.

Reduced motion is respected, focus is always visible, and everything works down
to a phone screen.

---

## What to build next

- **Card payments.** `payments` already models Square and PayPal; add a webhook
  that inserts a row and the rest of the chain fires on its own.
- **Customer accounts.** RLS policies for signed-in customers are already
  written — set `customers.auth_user_id` on signup and the portal works.
- **Membership billing.** `membership_plans` and the tier field are in place;
  wire the recurring charge and set `membership_renews`.
- **Inbound SMS.** `inbound_messages` is ready for a Plivo webhook, which turns
  Messages into a two-sided thread.
- **Route optimisation.** Route bookings already carry an address snapshot and
  cannot overlap each other.
