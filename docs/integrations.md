# RingDesk AI — Third-Party Integrations

No credentials, endpoints, or configuration values are included below — this documents *what* each integration does and *why* it was chosen, not how to configure it.

| Provider | Role | Notes |
|---|---|---|
| **Twilio** | Inbound/outbound calling, SMS | Signature-validated webhooks; validation logs mismatches but is deliberately never allowed to block a live call |
| **Retell AI** | Primary voice-agent orchestration | Verified via webhook signature header |
| **Groq** (`llama-3.3-70b`) | Live conversation engine | Chosen for inference speed — turn latency budget is 2–3.5 seconds total |
| **OpenAI** | Call-summary generation, intent classification | Used off the hot path, not in the live conversation loop |
| **ElevenLabs** | Primary text-to-speech synthesis | Called via raw REST, 5-second timeout before falling back |
| **Amazon Polly** (via Twilio `<Say>`) | Guaranteed TTS fallback | Ensures a caller is never met with silence |
| **Deepgram** | Speech-to-text | Called via raw REST |
| **Supabase** | Auth + Postgres database | Single admin-client singleton pattern; tenant isolation enforced at the data-access layer |
| **PayPal** | Primary billing | In-app checkout — deliberately no redirect/popup flow |
| **Stripe** | Secondary billing (card checkout) | Partially configured; signature-verified webhooks with idempotent event processing |
| **Google Calendar** | Appointment sync | Bookings write directly to the business's calendar |
| **Calendly** | Webhook-based appointment resolution | HMAC-SHA256 signature verification |
| **Resend** | Transactional email | Booking confirmations, notifications |
| **Telegram** | Internal alerting | Debug/ops notifications, allowlisted chat IDs |
| **Sentry** | Error monitoring | Plus an in-house Watchdog system for call-pipeline-specific observability |
| **Vercel** | Hosting/deployment | Serverless — informs the stateless-session design throughout the voice pipeline |

## Webhook security model

Every provider that sends inbound webhooks is verified before any database write:

| Provider | Verification |
|---|---|
| Retell | Custom signature header |
| Twilio | `validateRequest()` — logged, never blocking |
| Stripe | `stripe.webhooks.constructEvent()` |
| Calendly | HMAC-SHA256 signature header |

All of these additionally pass through a shared event-dedup gate before any mutation happens, keyed on each provider's native event ID.
