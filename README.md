# NabzChat - Architecture Case Study

> Multi-channel AI receptionist SaaS for SMBs. Production architecture for an active commercial venture, currently in private beta with first ground-truth customer in Dubai.

NabzChat answers WhatsApp messages, Telegram chats, and phone calls. It books appointments against a real Google Calendar, handles cancellations and reschedules in any of four languages, prevents double-bookings under race conditions, and notifies the business owner for approval. Built solo on a Linux VPS with direct API integrations, custom multi-tenant backend, and a self-managed operations stack.

**Status:** In active development. Private beta with one live business (Dubai-based beauty salon). Voice receptionist in test mode. Projected production launch Q3 2026.

This repository documents the **architecture and operations** - no source code is published here, as NabzChat is a proprietary commercial product.

---

## Why This Project Exists

Most AI booking systems break in ways that are obvious to anyone who has actually talked to one. They invent prices that don't exist on the menu. They confirm appointments without checking the calendar. They switch languages mid-conversation when the customer code-switches once. They double-book the same slot when two customers message simultaneously. They auto-confirm bookings the owner would have rejected.

Every architectural decision in NabzChat is a response to one of these failure modes. The system is built around the principle that the LLM is a **language interface**, not a source of truth. Calendars, brains, and human approval are the sources of truth.

---

## What NabzChat Does Today

### Customer-facing channels
- **WhatsApp Business API** - Meta App Review approved (whatsapp_business_messaging + whatsapp_business_management), business_management under review for Coexistence Mode
- **Telegram** - fully working with rich messaging, voice notes, images
- **Voice calls via Vapi.ai** - in test mode, end-to-end tested with live calls (call -> Luna answers -> checks calendar -> books -> notifies owner)
- **Instagram** - planned (same Luna brain over Instagram DM transport)

### Luna - the AI Secretary
- Claude Sonnet 4.5 with prompt caching (cached system prompts cut latency and cost ~70%)
- Per-message language detection (Persian, Arabic, Arabizi, English, Turkish)
- Voice transcription via OpenAI Whisper for Telegram voice messages
- Image understanding (customer sends photo - Luna sees it)
- Structured tool use for intent routing (no keyword matching)
- Per-business "Brain" - services, prices, durations, hours, FAQs, message templates
- Service duration awareness (Gel manicure = 45min, Acrylic = 90min, etc.)
- Configurable booking buffer per business (default 15 min)
- Gender clarification for businesses with both men's and women's services
- Owner handover when uncertain (`request_handover` tool)

### Booking system
- **Google Calendar as single source of truth** for availability
- Pre-fetched availability on every customer message (today + tomorrow)
- Pre-save validation right before INSERT (catches race conditions)
- Post-confirmation conflict scan (auto-rejects duplicate pending bookings)
- Two confirmation modes per business: **Manual** (owner taps confirm) and **Auto** (validated chain runs, owner notified after)
- Inline calendar picker for owner-initiated reschedule (month nav, hourly slots from Brain hours, Dubai timezone)
- In-place Telegram message editing (single message per booking, status updates replace previous state)
- Status buttons that evolve: Pending -> Confirmed -> Completed / Cancelled

### Cancellation and reschedule flows
- **Customer-initiated cancel** via `request_cancellation` tool -> owner approves/rejects
- **Customer-initiated reschedule** via `request_reschedule` -> owner approves new slot -> old calendar event deleted -> new event created
- **Owner-initiated cancel** on confirmed booking -> calendar event deleted -> customer notified
- **Owner-initiated reschedule** via inline calendar -> multi-slot offer -> customer picks via Luna -> owner notified
- All actions language-aware (customer hears it in their detected language)

### Multi-tenant architecture
- Row-level isolation via `business_id` on every table
- Tested with concurrent activity across two businesses - full isolation verified
- Each business has its own Brain, Luna personality, calendar connection, and message history
- Different staff and notification routing per business

---

## Tech Stack

| Layer | Tool | Why |
|-------|------|-----|
| Frontend | React 19 + Vite 6 + Tailwind v4 | Fast, modern, custom router (no React Router) |
| Backend | Node.js + Express (ES modules) | Synchronous-friendly for SQLite operations |
| Database | SQLite (better-sqlite3) | Synchronous API, sufficient for first 20-30 tenants |
| Auth | JWT + refresh token rotation | Refresh in httpOnly cookie, access in localStorage 15min |
| AI | Claude Sonnet 4.5 with prompt caching | Cached system prompts cut latency and cost ~70% |
| Voice (telephony) | Vapi.ai | $0.07/min all-in, ~600ms latency, no custom telephony |
| Voice (transcription) | OpenAI Whisper | For Telegram voice messages |
| Calendar | Google Calendar API (OAuth) | Single source of truth for availability |
| Messaging | Meta WhatsApp Business API | App Review approved |
| Bot framework | Telegram Bot API | Owner notifications + customer channel |
| Process management | PM2 (fork mode) | Production + monitor processes |
| Reverse proxy | Nginx + Cloudflare | TLS termination, DDoS protection |
| Backups | Cloudflare R2 (S3-compatible) | Hourly SQLite snapshots |
| DB admin | Adminer | Behind obscured URL, basic auth |
| Uptime monitoring | UptimeRobot | Multi-region health checks |
| Email | nodemailer + Zoho SMTP | Transactional emails |

---

## Key Architectural Decisions

### 1. The Brain - structured product knowledge, not vector store

Each business has a Brain: a structured database row with services, prices, durations, working hours, FAQs, and message templates. The LLM is given a JSON snapshot of the Brain on every call. There is no vector search, no RAG. Reasons:

- A salon menu is small (10-30 items). RAG is overkill.
- Structured data makes guardrails enforceable (Luna can never quote a price that isn't in `services[].price`).
- Owner updates Brain -> next message uses the new data immediately. No re-indexing.
- Debugging is trivial: read the JSON, see what Luna saw.

### 2. Calendar as the source of truth, not the database

Earlier iterations checked a database table of "pending bookings" before offering slots. This created false negatives - bookings the owner had already rejected still blocked slots because the table wasn't cleaned up reliably.

The new rule: **only Google Calendar can mark a slot as taken.** A booking is "real" only when the owner approves it (Manual Mode) or when all validations pass (Auto Mode), at which point the calendar event is created. Until then, the slot remains offerable.

This forced a second concern - the race condition.

### 3. Race condition prevention for double-booking

When two customers message at the same moment about the same slot, both see it as available (because no calendar event exists yet). The fix is two-layered:

- **Pre-save validation:** re-check the calendar one more time right before inserting the booking. Catches the common case.
- **Post-approval conflict scan:** when the owner confirms booking #1, the system immediately scans for other pending submissions with the same business + date + time and auto-rejects them, notifying those customers with alternative slots. Catches the simultaneous case.

### 4. Per-message language detection

Customers in Dubai mix Arabic, English, Arabizi (Arabic written in Latin letters), and sometimes Persian - sometimes within a single conversation. The system detects the language of **every incoming message** at the moment it arrives and routes Luna's response in that language.

This is paired with deterministic language detection helpers (`_hasPersian`, `_hasArabic`) for system-generated messages - tool responses, confirmations, error messages - so even non-LLM text reaches the customer in the language they wrote in.

### 5. Tool use over keyword matching for intent routing

When a customer says "I want to cancel my booking," the system doesn't pattern-match the text. Luna selects from a defined tool set:

- `request_cancellation` - customer wants to cancel an existing booking
- `request_reschedule` - customer wants to reschedule an existing booking
- `request_handover` - customer wants to talk to a human owner
- `confirm_reschedule` - customer accepts an offered new slot
- `reject_reschedule` - customer declines offered slots
- `add_booking_note` - customer adds a note (allergies, preferences) to their booking

Each tool has structured arguments and triggers a deterministic backend flow. The LLM decides *which* tool, but the *what happens next* is hard-coded. This is the boundary between LLM-driven and rule-driven that makes the system reliable.

### 6. Two confirmation modes - manual approval and validated auto

NabzChat supports two confirmation modes that the business owner toggles in settings.

**Manual Mode (default):** Luna collects details, runs availability checks, booking enters pending state, owner notified via Telegram with Confirm / Cancel / Reschedule buttons. Only after owner confirms does the calendar event get created.

**Auto Mode (validated):** Luna runs the full validation chain (availability, service exists in Brain, time inside working hours, slot not held), auto-confirms, creates the calendar event, then notifies the owner *after the fact* with cancel option. If any check fails, falls back to Manual.

The same applies to cancellations and reschedules - both have a manual-approval path and a validated auto path. New businesses start in Manual to build confidence in Luna's accuracy, then switch to Auto. **Auto Mode is not a separate code path - it's the same validation chain with a different confirmation step at the end.**

### 7. Multi-tenant row-level isolation

Every table that contains business data has a `business_id` column. Every query - without exception - filters by `business_id`. There is no shared state between tenants at the application level. Verified with concurrent activity across multiple businesses.

---

## DevOps and Operations

This is a solo-founder system, so every operational concern is also an architectural concern. The stack below was built incrementally as risks surfaced - not designed upfront, but evolved from real incidents.

### Environments
- **Production:** `app.nabzchat.tech` - main user-facing app
- **DB admin:** Adminer behind obscured URL with basic auth
- **Coming-soon page:** `nabzchat.tech` for public-facing brand

### Deploy pipeline
- Branch strategy: active dev branch with backup branches before risky changes (`backup/pre-security-fixes-*`, `backup/pre-redesign-*`)
- Deploy script enforces `git push origin -> git pull on server -> npm build -> PM2 reload`
- Health check at `/api/health` pings DB + AI provider + calendar before declaring deploy successful
- Push-then-deploy rule (deploy script does `git reset --hard`, so unpushed work would be lost - enforced as a habit and documented)

### Process supervision
- PM2 fork mode with two processes:
  - `nabzchat` - production API server (port 3000)
  - `nabzchat-monitor` - Python monitoring agent
- Auto-restart on crash
- PM2 logs streamed via tail for live debugging
- Process state survives server reboots (`pm2 startup` + `pm2 save`)

### Backups
- **Hourly SQLite snapshots** to Cloudflare R2 (S3-compatible object storage)
- Cron job triggers snapshot + upload + 7-day retention rotation
- Verified working end-to-end (this was previously flagged as the #1 infrastructure risk by an external review; now closed)

### Monitoring and alerting
- **UptimeRobot** multi-region health checks (catches outages from outside the host network)
- **Telegram deploy channel** receives:
  - Every uncaught backend exception
  - Every failed AI API call
  - Every deploy success/failure
  - Cron job failures
- **PM2 monitor process** watches process memory and restart count

### Security
- TLS via Cloudflare (free tier) - origin server only accepts traffic from CF IPs
- Refresh tokens rotated on use, stored in httpOnly cookie (XSS-resistant)
- Access tokens with 15-min expiry in localStorage (acceptable risk window)
- SSH on non-standard port with key-only auth
- All secrets in `.env` files, never in code, never in chat
- Cloudflare R2 keys scoped to the backup bucket only
- npm audit reviewed before each deploy

### Database operations
- Adminer for occasional inspection (behind obscure URL + basic auth + Cloudflare WAF)
- Migration approach: forward-only SQL, applied at deploy time
- Tested rollback: restore last hourly snapshot from R2

### Infrastructure
- **Hostinger KVM (Paris)** - Linux VPS as single point of compute
- **Cloudflare** - DNS, TLS, DDoS protection, R2 storage
- **Single-VPS risk acknowledged** - acceptable pre-launch, multi-region planned before 50 customers

---

## The Voice Receptionist (Vapi.ai)

Currently in test mode. Tested end-to-end with real phone calls.

**Flow:**
1. Customer calls Luna's Vapi number
2. Luna answers with the business's greeting (configurable in Brain)
3. Customer requests a service
4. Luna checks Google Calendar in real-time
5. Luna offers available slots
6. Customer picks a slot
7. Luna books -> calendar event created -> owner notified on Telegram
8. Customer hears confirmation: "I've booked you for Tuesday at 3pm."

The voice agent is the same Luna brain over a different transport. Same Brain access, same tool set, same calendar logic, same Telegram notifications. Vapi handles speech-to-text, text-to-speech, and telephony. Latency: ~600ms end-to-end.

**Why this matters:** Dubai SMBs don't use Telegram. WhatsApp requires Meta approval. Voice calls work everywhere, immediately, with no customer setup. Voice is the go-to-market channel.

---

## What I Learned

1. **LLMs should be language interfaces, not databases.** Every time I let Luna "know" something instead of looking it up, it eventually hallucinated.

2. **The calendar is the only thing that can say "no."** Database flags, pending tables, and in-memory locks all drifted out of sync with reality. Only the calendar stayed honest.

3. **Tool use is the boundary between LLM and code.** "LLM decides which tool, code decides what happens" is the reliability pattern.

4. **Multi-language is hard, per-message detection is simple.** Detecting each message at arrival and routing the response in the same language solved months of inconsistency complaints.

5. **Auto-confirmation is a mode, not a feature.** Building the validation chain for manual mode first means auto mode is just "trust the validation." Don't write two different code paths.

6. **One Brain per business beats one big knowledge base.** RAG is overkill for small product catalogs and makes debugging impossible.

7. **Synchronous SQLite is faster than I expected.** For the booking workload, `better-sqlite3` outperforms asynchronous patterns and removes a whole class of race conditions.

8. **DevOps is not a phase, it's the habit you build during the build.** Every incident (failed deploys, lost data, race conditions, alerting blind spots) added one item to the stack above. The stack didn't exist on day one - it accumulated from scars.

9. **Production tested with one real customer beats roadmap with ten planned ones.** Working on a real flow in a real business in a real city surfaces problems no amount of architecture planning would.

---

## What's Next

Roadmap from current state to first paying customer:

1. Voice receptionist out of test mode -> live (Vapi production setup with virtual UAE number)
2. Brain industry templates (Beauty Salon, Spa, Clinic, Real Estate) for faster onboarding
3. Luna confidence threshold + graceful fallback ("I'm not sure, let me get the owner")
4. UI redesign (current dashboard is functional, not sellable)
5. Owner/Platform admin panel for managing multiple businesses
6. Payment integration

Beyond first customer: spam protection, Instagram routing, CRM, analytics, broadcast campaigns, voice onboarding (Luna interviews the owner to build the Brain conversationally), multi-staff scheduling engine, PostgreSQL migration when SQLite saturates.

---

## Why No Source Code

NabzChat is a proprietary commercial product. The architecture is the transferable artifact - the implementation is environment-specific and will be sold or licensed, not open-sourced.

Available to discuss implementation in depth on request.

---

## Author

**Hamidreza Abolhassani**
Full-stack developer - AI Automation & SaaS
Email: hr.dev.abolhassani@gmail.com
GitHub: [github.com/abolhassani-dev](https://github.com/abolhassani-dev)
