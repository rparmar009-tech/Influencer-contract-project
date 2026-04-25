# Lovable build prompt — Influencer Contract Platform

> Paste everything below the divider into Lovable as your initial prompt. After Lovable scaffolds the first version, iterate by sending small follow-up prompts (one job or one screen at a time) — Lovable handles incremental changes much better than mega-prompts.

---

## Project: Influencer Contract & Escrow Platform

Build a web app that lets independent content creators send a simple, lightweight contract to a brand, collect upfront payment held in escrow, and auto-release the payment after delivery. Think Stripe-level simplicity — the creator should be able to draft and send a contract in under 60 seconds, and the brand should be able to review and pay without creating an account.

### Primary user
- **Content creator.** Not legal-savvy. Wants fast, frictionless deals. Biggest fear: not getting paid. Often working from a phone.

### Secondary user
- **Brand contact.** Marketing or social media manager. Reviews the contract via a public link. Should not be forced to sign up before paying.

---

## Tech stack

- React + Vite + TypeScript
- Tailwind CSS + shadcn/ui (Radix primitives)
- React Router for routing
- Supabase for auth (email + password, magic link), Postgres database, and file storage
- React Hook Form + Zod for form validation
- TanStack Query for server state
- Lucide-react for icons
- date-fns for time formatting and countdowns

Payments are out of scope for the first build — stub them with a "Pay via bank transfer" screen that just records a confirmation. Real payment integration will come later.

---

## Visual & UX design language

- **Aesthetic:** clean, modern, trust-forward. Borrow the visual restraint of Stripe and Linear. White / off-white backgrounds, generous whitespace, one accent color (use indigo `#4F46E5`), strong typographic hierarchy.
- **Type:** Inter for UI, monospace (JetBrains Mono) for contract numbers and amounts.
- **Components:** prefer shadcn/ui defaults (Button, Card, Dialog, Input, Toast, Badge, Separator, Progress, Stepper). Rounded-2xl cards, subtle shadows, no neon or glassmorphism.
- **Tone of voice:** direct, plain English, no legalese. Example labels: "Get paid", "Send contract", "Payment secured", "Awaiting delivery".
- **Mobile-first responsive.** Most creators are on mobile.
- **Motion:** subtle. Tailwind's `transition-all duration-200`. No hero animations.

### Three UX rules — enforce on every screen
1. **Money is always safe** — every status copy must reassure the user where their money is (Held in escrow / Released / Refunded).
2. **Status is always clear** — show a status badge on every page that involves a contract.
3. **Next action is always obvious** — exactly one primary CTA per page; secondary actions de-emphasized.

---

## Information architecture (routes)

Public:
- `/` — Landing page. Hero with value prop "Create a contract. Get paid upfront. No risk." Single primary CTA: "Create Contract" (no signup wall).
- `/new` — Contract creation form (works without an account; creator is prompted to claim the contract via email after creation).
- `/c/:contractId` — Public contract view (what the brand sees from the link). Shows summary, "Held in escrow" trust signal, two CTAs: Accept & Pay / Decline.
- `/pay/:contractId` — Bank transfer instructions + manual confirmation (stubbed payment).

Authenticated (creator):
- `/dashboard` — List of all contracts grouped by status, with a "+ New contract" button.
- `/dashboard/:contractId` — Single contract detail with status tracker, action buttons, and a chat-style activity log.
- `/dashboard/:contractId/submit` — Submit-delivery form (live URL + mandatory pre-approval checkbox + optional proof upload).
- `/login`, `/signup`, `/auth/callback`.

---

## Database schema (Supabase)

```sql
-- contracts
id uuid pk
public_token text unique         -- used for /c/:token public link
creator_id uuid references auth.users(id)  -- nullable until claimed
brand_name text not null
brand_email text not null
deliverables text not null
delivery_date date
price_cents integer not null
currency text default 'EUR'
status text not null             -- see status enum below
created_at timestamptz default now()

-- contract_events (audit log + activity feed)
id uuid pk
contract_id uuid references contracts(id) on delete cascade
type text                        -- created, sent, viewed, paid, submitted, accepted, disputed, resolved, refunded, cancelled
actor text                       -- creator | brand | system
metadata jsonb
created_at timestamptz default now()

-- payments (stubbed for MVP)
id uuid pk
contract_id uuid references contracts(id)
method text default 'bank_transfer'
amount_cents integer
status text                      -- pending | confirmed | refunded
confirmed_at timestamptz

-- delivery_submissions
id uuid pk
contract_id uuid references contracts(id)
live_url text not null
proof_path text                  -- supabase storage
approved_before_publish boolean not null
submitted_at timestamptz default now()

-- disputes
id uuid pk
contract_id uuid references contracts(id)
reason text                      -- not_as_agreed | late_delivery | other
comment text
opened_at timestamptz
creator_confirmed boolean
brand_confirmed boolean
resolution text                  -- pay_creator | refund_brand
resolved_at timestamptz
```

### Status enum
`draft` · `awaiting_payment` · `pending_payment` (failed retry) · `payment_secured` · `awaiting_approval` · `live_submitted` · `under_review` · `dispute_open` · `payment_released` · `cancelled` · `cancelled_refunded`

Render the status as a colored Badge everywhere a contract is shown. Map: secured / released = green; under_review = amber; dispute / refunded = red; everything else = neutral.

---

## Core flows

Build these in this order. Each one ends in a working, testable state.

### Job 1 — Create & share a contract

- `/` landing has a hero with value prop, an animated screenshot of the contract form, a "How it works" 3-step strip, and a single CTA → `/new`.
- `/new` is a single-screen form with five fields: Brand name, Brand email, Deliverables (textarea), Delivery date (optional date picker), Price (amount + currency dropdown defaulting to EUR). No legal/T&C inputs — display a small footnote: "Standard terms, escrow, and payment rules are auto-applied."
- On submit: create the contract row with status `awaiting_payment`, generate a `public_token`, send the brand an email with the public link, and show a success screen with: copy-to-clipboard, "Email it" (mailto), and "WhatsApp it" (`https://wa.me/?text=...`) buttons.
- Prompt the creator to claim the contract by entering their email → magic link signup. If they skip, store the contract anonymously and let them claim later.

### Job 2 — Brand reviews & pays

- `/c/:token` shows: creator name (or "Creator"), deliverables, delivery date, price big and bold, a "Held in escrow" badge with a tooltip explainer ("Your payment is held safely until the work is delivered and approved").
- Two buttons at bottom: primary "Accept & Pay" (→ `/pay/:token`), secondary "Decline".
- `/pay/:token` is a stubbed bank-transfer page: shows IBAN/reference (use placeholder values), a "Mark as paid" button (in real product this would be webhook-driven). Clicking it flips status to `payment_secured` and logs a `paid` event.
- If the brand never opens the link, the creator's dashboard shows resend / copy / send-reminder actions on the contract.

### Job 3 — Pre-approval & submit delivery

- On contract detail page (`/dashboard/:id`), once status is `payment_secured`, show a persistent banner: "To guarantee your payment, get the content approved by the brand before posting."
- Show a 3-step checklist: ☐ Content shared with brand · ☐ Approval received · ☐ Ready to publish. The checklist is informational (no toggles) — it's there to remind the creator of the order.
- "Submit Work" CTA opens `/dashboard/:id/submit`:
  - Live URL field (validated as a URL)
  - **Mandatory** checkbox: "I confirm this content was approved by the brand before going live" — submit button stays disabled until checked
  - Optional file upload for proof (image/pdf, max 10 MB, stored in Supabase Storage)
- On submit: status → `under_review`, start the 3-day timer (store `under_review_started_at`).

### Job 4 — Brand reviews delivery & payment releases

- After submission, the brand receives an email with a link to a review page. The page shows the live URL, a countdown ("Auto-release in: 2d 4h"), and two buttons: "Accept" (releases payment instantly) / "Raise issue" (opens dispute flow).
- A scheduled function (Supabase edge function or a Vercel cron) runs hourly and auto-releases any contract whose 3-day window has expired without action.
- On release: status → `payment_released`, log a `released` event, send celebratory email to creator. Show a "Payment released 🎉" toast on the dashboard.

### Job 5 — Dispute resolution (bilateral confirmation)

> Critical UX rule: the platform does **not** resolve creative disputes. Parties resolve externally and then **both** must confirm the outcome on-platform.

- "Raise Issue" opens a dialog: dropdown reason (Not as agreed / Late delivery / Other) + optional free-text comment + submit. On submit: status → `dispute_open`, payment frozen, both parties notified.
- Dispute screen for each party shows:
  - Clear copy: "Please resolve this directly with the other party and confirm the outcome here."
  - Two primary buttons: ✅ "Confirm Agreement" (release payment to creator) / ❌ "Mark as No Agreement" (cancel + refund).
  - Progress indicator: "Creator: ✅ Confirmed / Brand: ⏳ Pending" (or vice versa).
- Payment only releases when **both** parties click Confirm Agreement.
- **Deadlock rule:** after 10 days of no bilateral agreement, system prompts cancellation. If still no action, contract auto-cancels and refunds the brand.

### Job 6 — Edge cases

Handle on the contract detail page:
- **Payment fails:** error toast "Payment failed. Try another method."; status stays `pending_payment`.
- **Brand never opens link:** creator sees status `awaiting_payment` + actions [Resend link, Copy link, Send reminder email].
- **Creator made a mistake:** sent contracts are read-only; only available action is "Cancel Contract" → status `cancelled`. Cannot reactivate; create a new one.
- **Creator never delivers (deadline passes):** brand gets a "Report no delivery" CTA. On confirmation: full refund.
- **Fraud/abuse:** "Report issue" link in the contract footer for both parties. Click flags the contract internally (`flagged` boolean) and freezes payment. No investigation UI in MVP.

---

## Reusable components to build

- `<StatusBadge status="payment_secured" />` — color-coded chip.
- `<ContractCard contract={...} />` — summary card for the dashboard list.
- `<MoneyAmount cents={50000} currency="EUR" />` — formatted with locale.
- `<Countdown until={timestamp} />` — live-updating "2d 4h 12m" string.
- `<ActivityFeed contractId={...} />` — chat-style log of `contract_events`.
- `<TrustBadge />` — "Held in escrow" pill with info tooltip.
- `<StepChecklist items={[...]} />` — the 3-step pre-approval checklist.

---

## Out of scope for this build

Do **not** spend tokens on:

- Real payment processor integration (Stripe, etc.) — stub it.
- Italian fiscal validation, FatturaPA, P.IVA / Codice Fiscale — backend will handle.
- Localization beyond English — Italian copy comes later.
- Admin / ops dashboard.
- Two-factor auth, complex permissions.
- Mobile native apps.
- Public marketing pages beyond the single landing page.

Aim for a working MVP demo end-to-end in the first build, then iterate on polish.

---

## First-build acceptance checklist

When the first build is done, I should be able to:

1. Land on `/`, click Create Contract, fill the form, see a success screen with a copyable link.
2. Open that link in an incognito window — see the public contract, click Accept & Pay, mark as paid.
3. Sign up as the creator, see the contract in `/dashboard` with status `payment_secured`.
4. Click Submit Work, paste a URL, tick the approval checkbox, submit.
5. As the brand (in incognito), open the review email link and click Accept — see status flip to Payment Released.
6. As the brand, raise an issue instead — see the dispute screen with the bilateral-confirm UX.

Once that's working, I'll send follow-up prompts to polish each screen.
