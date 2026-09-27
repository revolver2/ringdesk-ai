# RingDesk AI — Engineering Highlights

## 1. A voice pipeline that is structurally incapable of going silent

The single hardest constraint on a phone product is that a caller can never be met with dead air — a slightly robotic voice is an acceptable degradation, silence is not. RingDesk enforces this with a 3-tier fallback: ElevenLabs (natural voice, primary) → Amazon Polly via Twilio's built-in `<Say>` (guaranteed available, secondary) → a hardcoded safety text string (tertiary, in case both providers fail). The ElevenLabs call has a hard 5-second timeout specifically so a slow third-party API can never turn into a stuck call. This is enforced as an "absolute rule" in the codebase's own operating contract — the fallback chain is explicitly called out as something no future change is allowed to remove.

## 2. Idempotent webhook processing under real-world retry pressure

Telephony and payment providers retry webhooks aggressively under any latency or ambiguous response. Every inbound webhook — Stripe, PayPal, Retell, Twilio, Calendly — is written to an `event_store` table (keyed on the provider's native event ID, with a database-level unique constraint) *before* any business-table mutation happens. A duplicate delivery is detected via that unique-constraint violation and returns HTTP 200 immediately with zero side effects. This is what makes it safe for a payment provider to retry a webhook five times without RingDesk ever double-charging, double-booking, or double-sending an SMS.

## 3. Stateless-by-design call sessions

Vercel's serverless functions are destroyed on cold start — any in-memory `Map` or `Set` used to track an in-progress call would silently lose state under normal serverless behavior. Every piece of call-session state (conversation history, extracted caller context, booking progress) is persisted to Postgres on every turn rather than held in process memory, which is what makes the voice pipeline correct under a deployment platform that offers no memory-persistence guarantee between requests.

## 4. Tenant isolation enforced at the data-access layer, not by convention

Rather than relying on every route handler to remember to scope its queries by `user_id`, the codebase funnels all database access through exactly two functions — `systemScoped()` for system/cron operations and `userScoped(id)` for anything tenant-specific — and bans direct admin-client instantiation everywhere else. This turns "did someone forget to filter by tenant" from a code-review judgment call into a structural constraint: there's exactly one place that kind of bug could hide, and it's a small, well-known one.

## 5. One billing gate, deliberately

`lib/plan-access.ts` is documented as "the only source of truth" for plan/feature access, with an explicit rule against ever adding a second gate. This is a direct response to a common SaaS failure mode — two independently-maintained billing checks drifting apart over time, resulting in either a paying customer getting incorrectly blocked or a feature leaking for free. Centralizing it in one function makes that class of bug much harder to introduce by accident.

## 6. A booking flow that guards against its own double-execution

`finalizeCall()` — which sends the SMS confirmation, creates the calendar event, and logs usage — is only allowed to fire once per call. The session's `callRecordId` is set to `null` immediately after the first successful booking, and a `bookingConfirmed` flag prevents the same call from triggering a second SMS/calendar-write if the conversation continues past the booking moment (e.g., "is there anything else I can help with?"). This is a small detail that matters a lot in production: without it, a longer post-booking conversation would risk sending a customer two confirmation texts for one appointment.
