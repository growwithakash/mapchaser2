# Map Chaser — Compliance Checklist

**Status: pre-launch draft.** This document describes what the current codebase
technically supports and what still needs professional review. It is a working
checklist, not a certification. **Map Chaser is not claimed to be "100% legally
compliant."** Items marked ⚠️ REQUIRES PROFESSIONAL REVIEW must be checked by a
qualified lawyer, accountant, and/or licensed payment partner for your
jurisdiction before accepting real users, uploads, or payments.

---

## 1. No prohibited mechanics (by design)

Implemented in the database schema and trigger logic (`handle_upload_status_change`,
`platform_settings`):

- No gambling, betting, lottery, or paid random-chance mechanics anywhere in the
  reward flow. Points are earned only for **approved, original uploads** —
  never for chance, wagering, or engagement metrics.
- Reward values (`photo_points`, `video_points`, `conversion_rate_points_per_unit`,
  `min_redeem_points`, upload limits) live in `platform_settings`, not hard-coded,
  and are readable by any client before upload/redeem.
- All point math runs **server-side** via a `SECURITY DEFINER` Postgres trigger —
  no client can set its own point value.
- No income is promised anywhere in the UI copy ("Approved and Reward unlock
  once a reviewer checks your upload — never before").

⚠️ REQUIRES PROFESSIONAL REVIEW: confirm your specific reward/conversion design
doesn't inadvertently resemble a regulated financial product in the
jurisdictions you serve.

## 2. UGC moderation

**Implemented:**
- Every upload defaults to `status = 'pending'`. Points are credited only on
  an explicit transition to `'approved'` — enforced by RLS (only `is_admin()`
  can update `uploads.status`) plus the trigger, not by client trust.
- `rejection_reason` is a constrained enum (`not_original`, `low_quality`,
  `not_travel_nature`, `unsafe_content`, `policy_violation`, `other`).
- Repeat-violation handling: a `policy_violation` rejection increments
  `profiles.strikes`; 3 strikes sets `profiles.suspended = true`, which an
  RLS policy uses to block further uploads from that account.
- User reporting: `reports` table + RLS (authenticated, non-suspended users
  can file; only admins can read/resolve). **UI pending** — no public gallery
  is wired to live data yet, so there's nothing yet for a user to report from
  inside the app. Build the report button alongside the public gallery feature.
- Full audit trail: every status change is written to `audit_logs`
  (actor, action, before/after, timestamp), readable only by admins.

**Implemented — real automated moderation (`app/api/moderate`):**
- Every photo upload is sent to Claude's vision API for three real checks:
  travel/nature relevance, quality (corrupt/blank/blurry), and safety
  (explicit, violent, hateful content). The model also returns its own
  recommendation and reasoning, stored in `moderation_results` for admins to
  see — never shown to the uploader or other users.
- Real on-platform duplicate detection: a perceptual hash (aHash) of every
  photo is computed server-side (no third-party API) and compared against
  every previous upload's hash. A close match is **never auto-rejected** —
  it's routed to manual review, since perceptual hashing can false-positive.
- **Fail-safe by construction**: if `ANTHROPIC_API_KEY` is missing, the
  Claude call fails, the response can't be parsed, or the file can't be
  downloaded, the upload goes to `manual_review` and the failure is logged —
  it is never auto-approved by default or by error.
- Honesty limits, stated plainly to the model and enforced in code: the AI
  is explicitly told it cannot judge originality against the wider internet
  from a single image, and is never allowed to factor that into its
  recommendation. "Reject" is only trusted for high-confidence safety or
  relevance failures; anything uncertain routes to a human.
- **Video uploads**: a real keyframe pipeline now runs (ffmpeg, bundled as a
  static binary, samples 3 frames per video), and each frame goes through the
  same relevance/quality/safety/duplicate checks as photos. This is honestly
  a *sample*, not exhaustive coverage — three frames cannot see everything in
  a video. If frame extraction fails for any reason (unsupported codec,
  environment issue, timeout), the video fails safely to manual review rather
  than being guessed at. This has been tested against a synthetic video in
  the build environment, but **not yet verified on live Vercel
  infrastructure** — ffmpeg-in-serverless is a known-finicky pattern; confirm
  it behaves the same way after your own deployment.
- **Not yet covered**: duplicate matching against *other* platforms/the open
  internet (would need a paid reverse-image API — see the original Copy
  Detection section of the product brief).
- Appeal process for rejected uploads — not built. Recommended: reuse the
  `reports`-style pattern, or let a user re-submit with context.

⚠️ REQUIRES PROFESSIONAL REVIEW: moderation SLAs and what counts as
"reasonable effort" for a platform your size, under the laws you operate in
(e.g. IT Rules 2021 due-diligence obligations in India, DSA in the EU, etc.).

## 3. Copyright

**Implemented:**
- `copyright_claims` table, open to anonymous submission (claimants need no
  account) via `/copyright`, admin-only to read/resolve.
- Uploaders must affirmatively confirm ownership/permission before submitting
  (checkbox in the upload flow).

**Not yet implemented:**
- Automatic hiding of content while a claim is under review (currently
  requires an admin to manually set status) — add this once the admin panel
  is wired to the live database.
- A published Copyright Policy inside the live app (the policy text currently
  lives only in the separate static marketing site, not in this codebase).

⚠️ REQUIRES PROFESSIONAL REVIEW: a formal DMCA-style (or local equivalent)
counter-notice process before commercial launch.

## 4. Privacy / data protection

**Implemented:**
- Minimal collection: email, auth metadata, upload content/metadata,
  wallet/transaction records — nothing beyond what the service needs.
- Row Level Security on every table: users can only read their own profile,
  wallet, uploads, and transactions (plus publicly approved uploads once a
  gallery is built); sensitive moderation signals stay admin-only.
- Self-serve account deletion (`/api/account/delete`) that removes the auth
  user and cascades to all owned rows (uploads, wallet, transactions,
  reports) via foreign-key `ON DELETE CASCADE`.
- Secrets hygiene: the service-role key is documented as server-only in
  `.env.example` and is never referenced from client components.

**Not yet implemented:**
- A published Privacy Policy inside the live app (again, currently only in
  the separate static site — bring that text into this app before launch).
- Data export ("download my data") — deletion exists, export does not yet.

⚠️ REQUIRES PROFESSIONAL REVIEW: applicability and specific obligations under
India's DPDP Act 2023 and any other country you serve (consent language,
grievance officer appointment, breach notification timelines, cross-border
transfer rules).

## 5. Rewards & consumer protection

- Conversion rate, minimum redemption, and limits are all queryable from
  `platform_settings` before a user acts — no hidden terms.
- No "earn money easily" language anywhere in the UI copy.
- Redemption statuses (`pending/processing/completed/rejected/cancelled`)
  are modelled but **not yet wired to a real payment provider** — redemption
  is schema-ready, not live. Do not advertise withdrawals as available until
  a licensed payment partner is integrated.

⚠️ REQUIRES PROFESSIONAL REVIEW: consumer-protection disclosure requirements
for the specific reward/redemption design, in each jurisdiction served.

## 6. Payments, withdrawals & tax

**Not yet implemented** — this phase intentionally stops short of money
movement. Before enabling real redemption:
- Integrate a licensed payment/payout provider (e.g. Razorpay/Stripe)
  server-side only.
- Decide and implement TDS/tax-withholding obligations, if any, for reward
  payouts in your jurisdiction.
- Add KYC collection appropriate to payout size and local regulation.

⚠️ REQUIRES PROFESSIONAL REVIEW: this entire section, before any real payout
is made. Do not skip.

## 7. Advertising

No advertising is integrated in this codebase. If/when ads (e.g. Google
AdSense) are added later:
- Ads must only appear on pages/content compliant with the ad network's
  policies (nothing age-restricted, infringing, or unsafe).
- Never reward users (points, badges, anything) for viewing or clicking an
  ad — this is a bannable violation under most ad network policies and is
  explicitly excluded from this design.
- Ads must be visually distinguishable from platform content, never styled
  to look like an upload, a notification, or navigation.

## 8. Age-related protections

**Implemented (pending one manual database step):** a minimum-age gate
(default 18, stored in `platform_settings.min_signup_age`) applies because
rewards can be redeemed for real money.
- Email signup collects a date of birth; an underage signup is rejected by a
  database trigger, not just by the form.
- Google sign-in provides no birth date, so those users are sent to
  `/complete-profile` before using the app. An underage date of birth
  automatically suspends the account (set by a database trigger, not by
  client code), and the date cannot be changed afterwards without support.
- Uploading and redemption are blocked at the database (RLS) level for any
  account with no date of birth or a suspended account, so bypassing the UI
  does not bypass the gate.
- **The migration is not applied yet.** Run `supabase/age_gate_and_hardening.sql`
  in the Supabase SQL Editor before deploying this version. It was written
  while the project was paused and has not been run against the live database.

**Limits, stated plainly:** this is a self-declared date of birth, not
verified identity. Someone can lie, or sign up again with another email.
Real age verification (ID checks) is out of scope here.

⚠️ REQUIRES PROFESSIONAL REVIEW: the right minimum age for your market and
payout model, and any parental-consent rules below it.

## 9. Content safety (explicitly out of scope by design)

The schema and UI never facilitate: sexual/explicit content, graphic
violence, hate/harassment, illegal activity, spam, or fraud. The
`rejection_reason` enum and `unsafe_content` / `policy_violation` categories
exist specifically so a human reviewer can act on these before any point is
credited. No automated detection is wired in yet (see Section 2) — human
review is the only safeguard until it is.

## 10. Policy documents

| Document | Status |
|---|---|
| Terms & Conditions | Drafted in the separate static marketing site only — **not yet in this app** |
| Privacy Policy | Same — drafted separately, needs to be brought into this app |
| Community Guidelines | Same |
| Copyright Policy | Same — and the live `/copyright` intake form should link to it |
| Reward Policy | Same |
| Withdrawal Policy | Same — update once a real payment provider is integrated |
| Complaint / grievance handling | `reports` + `copyright_claims` tables exist; a published grievance-officer contact (if required in your jurisdiction) is not yet in this app |

⚠️ REQUIRES PROFESSIONAL REVIEW: every policy document above, before
commercial launch, by a qualified lawyer for each jurisdiction served.

## 11. Fail-safe behaviour

Designed principle, applied consistently: **if a required check or service
is unavailable, nothing is auto-approved and no reward is credited.**
Concretely: uploads default to `pending`; only an authenticated admin action
(or a future, explicitly-wired automated check) can move status forward;
account deletion returns a clear error instead of partially deleting data if
the service-role key isn't configured; RLS denies by default on every table
unless a policy explicitly allows the action.

## 11b. Security hardening found during review

While adding the age gate, several permissive policies from earlier phases
were found and are fixed in `supabase/age_gate_and_hardening.sql`:
- Any signed-in user could edit any column of their own profile (including
  role, suspension and strikes). Now only `full_name` and a one-time date of
  birth are user-editable; the rest needs an admin or the service role.
- A user could insert an upload already marked `approved`, publishing
  unmoderated content. Inserts must now be `pending` with zero points.
- Reports and copyright claims could be filed pre-resolved; notifications
  could be edited beyond `read_at`. All restricted.
- `daily_upload_limit` and `monthly_upload_limit` existed but were never
  enforced; they now are, which also caps AI-moderation cost abuse.
- The storage bucket now limits file size (50 MB) and types.
- `/api/moderate` now requires the uploader or an admin, and a failed
  duplicate check can no longer lead to auto-approval.

## 11c. Admin access model (revised after runtime testing)

Runtime testing found that the original admin layout redirected every /admin page
(including the 2FA setup page itself) to 2FA setup, so enrolment could never complete.
Fixed: the 2FA pages now live in their own route group. Admin access is now checked in
three places, not only the layout: the layout, every admin page, and every admin server
action (`lib/auth/require-admin.ts`), always before any side effect such as sending an
email. Database RLS remains the final backstop. The public copyright-acknowledgment
action only sends mail for a claim just filed from that address, rate-limited per address.

**Not tested end to end:** real sign-in, upload, AI moderation, email delivery and the
admin panel with a live 2FA session were not exercised against a live deployment (the
build environment cannot reach Supabase or hold your API keys). The database rules were
tested directly (rolled-back SQL tests); the app was tested for build, routing, redirects
and anonymous API rejection. Test the full flow yourself after deploying.

## 12. Known residual dependency advisory

**Update:** this app now runs on Next.js 16.4.2 and React 19.3.1 (upgraded
from 14.2.35 / React 18). The `postcss` advisory described in earlier drafts
of this document, and the critical/high-severity Next.js CVEs that motivated
the upgrade, are now resolved — confirmed via a clean `npm audit` and a full
production build with zero errors and zero warnings. The deprecated
`middleware.ts` convention was migrated to `proxy.ts` via Next's official
codemod.

One `moderate` severity advisory remains: `jimp` (used for perceptual-hash
duplicate detection) depends on a `file-type` version with an ASF-parser
infinite-loop DoS. This platform only ever decodes JPEG/PNG images it
receives from its own authenticated uploaders — it never parses arbitrary
ASF/WMA files — so the realistic exposure is low. A fix exists only via a
major `jimp` v1 rewrite (a breaking API change), not yet done; revisit if
jimp's maintainers backport a fix, or before significantly scaling uploads.

---

### One-line summary for stakeholders

The technical architecture is built so that nothing ships a false promise:
no fake moderation, no fake payouts, no silent data collection, and no path
that credits a reward without a real approval. What's left before commercial
launch is largely **legal, tax, and payment-provider work**, not application
logic — and that work needs professional sign-off, not an AI's judgment.
