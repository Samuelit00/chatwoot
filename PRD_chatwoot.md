# PRD — Chatwoot Implementation for Customer Chat Management

**Project:** logicfy-chatwoot (standalone service)
**Date:** May 15, 2026
**Author:** Jhon Sanchez
**Assigned Developer:** Samuel Moncayo
**Context:** This project deploys a self-hosted Chatwoot instance on Railway to provide logicfy customers with a professional chat UI for WhatsApp conversations. It supports both WhatsApp Cloud API (native) and Evolution API Baileys sessions.

---

## 1. Executive Summary

logicfy needs a customer-facing chat interface where users can manage WhatsApp conversations with their customers through a professional support inbox. **Chatwoot** (open source, self-hosted) provides this capability out of the box.

This deployment must support **500 accounts** (logicfy customers), each with up to **10 agent seats** and approximately **3 concurrent conversations** per account. The system must support two WhatsApp connection methods:

1. **WhatsApp Cloud API** — connected natively in Chatwoot via Embedded Signup (full 24-hour window management, templates, read receipts)
2. **Evolution API Baileys** — connected through the Evolution API Chatwoot integration (no Meta restrictions, QR-based)

### Objectives

1. Deploy Chatwoot on Railway with PostgreSQL (pgvector), Redis, and Sidekiq
2. Configure WhatsApp Cloud API with Embedded Signup (requires Facebook App creation)
3. Configure Evolution API integration for Baileys-based sessions
4. Set up Cloudflare R2 for media storage and Resend for transactional email
5. Host at `chat.logicfy.co` with HTTPS
6. Enable all Chatwoot features: CSAT, canned responses, automations, reports, knowledge base

### Out of Scope (Future Phases)

- logicfy platform integration (SSO, automated account provisioning, AI handoff)
- White-labeling / custom branding
- Instagram and Facebook Messenger channels
- Bulk messaging / campaign features

---

## 2. Technical Context

### 2.1 What is Chatwoot?

Chatwoot is an open-source customer engagement platform built with **Ruby on Rails**. It provides a real-time chat inbox for support teams.

| Aspect | Details |
|---|---|
| Repository | [github.com/chatwoot/chatwoot](https://github.com/chatwoot/chatwoot) |
| Language | Ruby on Rails + Vue.js |
| Database | PostgreSQL 14+ (with pgvector extension) |
| Cache/Queue | Redis 6+ |
| Background Jobs | Sidekiq |
| Docker Image | `chatwoot/chatwoot:v3.x.x-ce` |
| License | MIT (Community Edition) |

### 2.2 Why Chatwoot?

| Requirement | Chatwoot Capability |
|---|---|
| Chat inbox UI | ✅ Full-featured agent inbox with real-time messaging |
| Multi-tenancy | ✅ Accounts with isolated workspaces |
| Agent management | ✅ Roles (admin/agent), teams, assignment rules |
| WhatsApp Cloud API | ✅ Native integration with Embedded Signup, 24h window, templates |
| Evolution API | ✅ Supported via Evolution API's native Chatwoot integration |
| Self-hosted | ✅ Full control, no per-seat SaaS costs |
| CSAT & Reports | ✅ Built-in satisfaction surveys and analytics |

---

## 3. System Architecture

### 3.1 High-Level Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Railway Platform                              │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │  Chatwoot     │  │  Chatwoot     │  │  Evolution API        │  │
│  │  Web (Puma)   │  │  Sidekiq      │  │  (separate project)   │  │
│  │  Port 3000    │  │  Workers      │  │  Port 8080            │  │
│  └──────┬────────┘  └──────┬────────┘  └───────────┬───────────┘  │
│         │                  │                       │              │
│         └──────────┬───────┘                       │              │
│                    │                               │              │
│         ┌──────────┴──────────┐                    │              │
│         │                     │                    │              │
│    ┌────┴─────┐  ┌────────────┴┐                   │              │
│    │PostgreSQL │  │    Redis    │                   │              │
│    │(pgvector) │  │             │                   │              │
│    └──────────┘  └─────────────┘                   │              │
│                                                     │              │
└─────────────────────────────────────────────────────┘              │
                                                                     │
         ┌───────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│                  WhatsApp Connections                     │
│                                                          │
│  Path A: Cloud API (Native)         Path B: Baileys      │
│  ┌─────────────────────┐           ┌──────────────────┐  │
│  │ Chatwoot ──► Meta   │           │ Chatwoot ──►     │  │
│  │ (Embedded Signup)   │           │ Evolution API ──►│  │
│  │ • 24h window mgmt   │           │ Baileys ──►     │  │
│  │ • Template selector │           │ WhatsApp Web    │  │
│  │ • Read receipts     │           │ • No restrictions│  │
│  └─────────────────────┘           └──────────────────┘  │
└─────────────────────────────────────────────────────────┘

External Services:
  • Cloudflare R2 ── Media/file storage (S3-compatible)
  • Resend ── Transactional email (SMTP)
  • Meta Graph API ── WhatsApp Cloud API
```

### 3.2 Railway Services

| Service | Image / Type | Purpose | Estimated Resources |
|---|---|---|---|
| **Chatwoot Web** | `chatwoot/chatwoot:v3.10.x-ce` | Rails app server (Puma) | 2-4 GB RAM, 2 CPU |
| **Chatwoot Sidekiq** | Same image, different command | Background job processing | 2-4 GB RAM, 2 CPU |
| **PostgreSQL** | Railway Plugin (pgvector) | Data persistence | 2-4 GB RAM, 50+ GB SSD |
| **Redis** | Railway Plugin | Cache, Sidekiq queue, ActionCable | 512 MB - 1 GB RAM |

> **⚠️ Critical:** PostgreSQL **must** have the `pgvector` extension enabled. Standard PostgreSQL will cause `PG::FeatureNotSupported` errors. Use Railway's pgvector template or install the extension manually.

### 3.3 Capacity Planning for 500 Accounts

| Metric | Estimate |
|---|---|
| Total accounts | 500 |
| Max agents per account | 10 |
| Total potential agents | 5,000 |
| Concurrent active accounts (30%) | 150 |
| Concurrent conversations (150 × 3) | 450 |
| Peak concurrent agents (estimated) | 300-500 |

#### Resource Scaling Strategy

| Phase | Accounts | Web Workers | Sidekiq Workers | PostgreSQL | Redis |
|---|---|---|---|---|---|
| Phase 1 — MVP | 0-50 | 1 instance (2 GB) | 1 instance (2 GB) | 1 GB RAM | 512 MB |
| Phase 2 — Growth | 50-200 | 1 instance (4 GB) | 1 instance (4 GB) | 2 GB RAM | 512 MB |
| Phase 3 — Scale | 200-500 | 2 instances (4 GB each) | 2 instances (4 GB each) | 4 GB RAM | 1 GB |

---

## 4. WhatsApp Integration — Two Paths

### 4.1 Path A: WhatsApp Cloud API (Native in Chatwoot)

This is the **recommended path** for users who want full official WhatsApp features.

#### How It Works

1. Super Admin configures the Facebook App credentials in Chatwoot
2. Users go to **Settings → Inboxes → Add Inbox → WhatsApp**
3. Users select **"Connect with WhatsApp Business (Embedded Signup)"**
4. A Meta popup appears — user authenticates, selects their Business Account, verifies phone number
5. Chatwoot automatically receives the API token and configures the webhook
6. The inbox is ready — full Cloud API features available

#### Features Available

| Feature | Supported | Notes |
|---|---|---|
| 24-hour conversation window | ✅ Yes | Chatwoot tracks window, blocks free-form when expired |
| Template messages | ✅ Yes | Agent selects from pre-approved templates when window is closed |
| Template creation | ❌ No | Must be created in Meta Business Manager |
| Read receipts | ✅ Yes | Blue checkmarks visible in conversation |
| Media messages | ✅ Yes | Images, video, audio, documents |
| Delivery status | ✅ Yes | Sent, delivered, read indicators |
| Contact profiles | ✅ Yes | Name, phone number |
| Embedded Signup (OAuth popup) | ✅ Yes | Seamless onboarding for users |

#### Required Facebook App Setup

Before Embedded Signup works, logicfy must create and configure a Facebook App:

| Step | Action |
|---|---|
| 1 | Go to [developers.facebook.com](https://developers.facebook.com/) → Create App |
| 2 | Select app type: **Business** |
| 3 | Add the **WhatsApp** product to the app |
| 4 | Add the **Facebook Login for Business** product |
| 5 | In Facebook Login for Business → Configuration, create a config with variation **"WhatsApp Embedded Signup"** |
| 6 | Note the **App ID**, **App Secret**, and **Configuration ID** |
| 7 | Submit the app for **App Review** (required for production Embedded Signup) |

#### Chatwoot Environment Variables for Cloud API

```env
# Set via Super Admin panel at /super_admin/app_config?config=whatsapp_embedded
WHATSAPP_APP_ID=your-facebook-app-id
WHATSAPP_CONFIGURATION_ID=your-embedded-signup-config-id
WHATSAPP_APP_SECRET=your-facebook-app-secret
```

### 4.2 Path B: Evolution API Baileys (via Integration)

This path is for users who connect via QR code (unofficial WhatsApp Web protocol).

#### How It Works

1. User creates a WhatsApp instance in Evolution API (QR code scan)
2. In Evolution API Manager → Integrations → Chatwoot, configure:
   - Chatwoot URL: `https://chat.logicfy.co`
   - Account ID: the user's Chatwoot account ID
   - Token: the user's Chatwoot access token
   - Enable "Auto Create": creates an inbox in Chatwoot automatically
3. Evolution API syncs messages bidirectionally with Chatwoot
4. All messages appear in the Chatwoot inbox in real-time

#### Features Available

| Feature | Supported | Notes |
|---|---|---|
| Free-form messaging (anytime) | ✅ Yes | No 24-hour restriction |
| Media messages | ✅ Yes | All types supported |
| Message reactions | ✅ Yes | Via Evolution API |
| Message deletion | ✅ Yes | Bidirectional sync |
| Read receipts | ✅ Yes | Via Evolution API |
| Contact import | ✅ Yes | Optional, configurable |
| Message history import | ✅ Yes | Optional, configurable |
| 24-hour window management | ❌ N/A | Not needed — Baileys has no restrictions |
| Template messages | ❌ N/A | Not needed — Baileys has no restrictions |

#### Evolution API Environment Variables (for Chatwoot)

```env
# In Evolution API's .env
CHATWOOT_ENABLED=true
CHATWOOT_MESSAGE_READ=true
CHATWOOT_MESSAGE_DELETE=true
CHATWOOT_BOT_CONTACT=true
```

### 4.3 Comparison: Cloud API vs. Baileys via Evolution API

| Aspect | Cloud API (Native) | Baileys (Evolution API) |
|---|---|---|
| **Connection method** | OAuth popup (Embedded Signup) | QR code scan |
| **24h window tracking** | ✅ Full UI support | N/A (no restrictions) |
| **Template messages** | ✅ Template selector in UI | N/A (no restrictions) |
| **Ban risk** | ✅ None (official) | ⚠️ Medium (unofficial) |
| **Cost** | Meta per-conversation pricing | Free (self-hosted) |
| **Setup complexity** | Medium (Facebook App required) | Low (just QR scan) |
| **Requires Evolution API** | ❌ No | ✅ Yes |
| **Best for** | Businesses needing compliance | Small businesses, testing |

---

## 5. External Services Configuration

### 5.1 Cloudflare R2 (Media Storage)

Chatwoot uses S3-compatible storage for file uploads and media attachments.

```env
ACTIVE_STORAGE_SERVICE=s3_compatible
STORAGE_BUCKET_NAME=logicfy-chatwoot-media
STORAGE_ACCESS_KEY_ID=your-r2-access-key
STORAGE_SECRET_ACCESS_KEY=your-r2-secret-key
STORAGE_REGION=auto
STORAGE_ENDPOINT=https://your-account-id.r2.cloudflarestorage.com
```

> **⚠️ Important:** Cloudflare R2 requires checksum configuration to prevent silent upload failures. The `config/storage.yml` in the Chatwoot container may need to be updated with `request_checksum_calculation: 'when_required'` and `response_checksum_validation: 'when_required'`. Verify this during deployment.

#### R2 Bucket CORS Configuration

```json
[
  {
    "AllowedOrigins": ["https://chat.logicfy.co"],
    "AllowedMethods": ["GET", "PUT", "POST", "DELETE"],
    "AllowedHeaders": ["*"],
    "MaxAgeSeconds": 3600
  }
]
```

### 5.2 Resend (Transactional Email)

Chatwoot requires SMTP for agent invitations, password resets, and notifications.

```env
SMTP_ADDRESS=smtp.resend.com
SMTP_PORT=587
SMTP_USERNAME=resend
SMTP_PASSWORD=your-resend-api-key
SMTP_AUTHENTICATION=plain
SMTP_ENABLE_STARTTLS_AUTO=true
MAILER_SENDER_EMAIL=chat@logicfy.co
```

> **⚠️ Important:** These SMTP variables must be set on **both** the Web and Sidekiq services. Sidekiq processes email delivery jobs — if it's not configured, emails will fail silently.

### 5.3 Custom Domain

| Domain | Service | Purpose |
|---|---|---|
| `chat.logicfy.co` | Chatwoot Web | User-facing chat application |

Configure via Railway's custom domain settings. Railway provides automatic SSL via Let's Encrypt.

---

## 6. Chatwoot Environment Variables (Complete)

### 6.1 Core Application

```env
# ─── Application ───
SECRET_KEY_BASE=<generate with: openssl rand -hex 64>
FRONTEND_URL=https://chat.logicfy.co
RAILS_ENV=production
NODE_ENV=production
RAILS_LOG_LEVEL=info
LOG_LEVEL=info

# ─── Database ───
DATABASE_URL=postgresql://user:password@host:5432/chatwoot_production
POSTGRES_HOST=<railway-internal-host>
POSTGRES_PORT=5432
POSTGRES_DATABASE=chatwoot_production
POSTGRES_USERNAME=chatwoot
POSTGRES_PASSWORD=<strong-password>

# ─── Redis ───
REDIS_URL=redis://host:6379

# ─── Storage (Cloudflare R2) ───
ACTIVE_STORAGE_SERVICE=s3_compatible
STORAGE_BUCKET_NAME=logicfy-chatwoot-media
STORAGE_ACCESS_KEY_ID=<r2-access-key>
STORAGE_SECRET_ACCESS_KEY=<r2-secret-key>
STORAGE_REGION=auto
STORAGE_ENDPOINT=https://<account-id>.r2.cloudflarestorage.com

# ─── Email (Resend) ───
SMTP_ADDRESS=smtp.resend.com
SMTP_PORT=587
SMTP_USERNAME=resend
SMTP_PASSWORD=<resend-api-key>
SMTP_AUTHENTICATION=plain
SMTP_ENABLE_STARTTLS_AUTO=true
MAILER_SENDER_EMAIL=chat@logicfy.co

# ─── WhatsApp Cloud API (Embedded Signup) ───
WHATSAPP_APP_ID=<facebook-app-id>
WHATSAPP_CONFIGURATION_ID=<embedded-signup-config-id>
WHATSAPP_APP_SECRET=<facebook-app-secret>

# ─── Features ───
ENABLE_ACCOUNT_SIGNUP=true
DIRECT_UPLOADS_ENABLED=true
```

### 6.2 Sidekiq Worker

Uses the **same image** as Chatwoot Web but with a different entrypoint:

```
Command: bundle exec sidekiq -C config/sidekiq.yml
```

All environment variables from 6.1 must be shared with the Sidekiq service (DATABASE_URL, REDIS_URL, SMTP_*, STORAGE_*, etc.).

---

## 7. Account & Agent Structure

### 7.1 Recommended Limits Per Account

| Setting | Recommended Limit | Rationale |
|---|---|---|
| **Max agents per account** | 10 | Covers most SMB teams (owner + managers + support staff). Prevents resource abuse |
| **Max inboxes per account** | 5 | Multiple WhatsApp numbers + future channels (Instagram, email) |
| **Max teams per account** | 5 | Sales, Support, Billing, etc. |

### 7.2 Agent Roles

| Role | Permissions |
|---|---|
| **Administrator** | Full account access: manage agents, inboxes, settings, integrations, billing |
| **Agent** | Handle conversations, use canned responses, view reports for assigned conversations |

### 7.3 Account Creation Flow (This Phase)

Since logicfy integration is out of scope, accounts will be created manually:

1. Super Admin creates accounts via Chatwoot Super Admin panel (`/super_admin`)
2. Super Admin invites the account administrator via email
3. Account admin logs in, sets up their inbox (WhatsApp Cloud API or Evolution API)
4. Account admin invites their agents via email

> **Future Phase:** logicfy will automate this flow via the [Chatwoot Platform API](https://www.chatwoot.com/docs/product/others/platform-api), creating accounts and agents programmatically when a user signs up.

---

## 8. Chatwoot Features Checklist

All features below should be enabled and verified during deployment:

| Category | Feature | Priority |
|---|---|---|
| **Conversations** | Real-time messaging (text, media) | ✅ Critical |
| | Conversation assignment (manual + auto) | ✅ Critical |
| | Conversation labels and filters | ✅ High |
| | Private notes between agents | ✅ High |
| | Reply-to / quoted messages | ✅ High |
| | Typing indicators | ✅ High |
| **Agents** | Agent presence (online/offline/busy) | ✅ High |
| | Agent teams | ✅ High |
| | Canned responses (quick replies) | ✅ High |
| | Keyboard shortcuts | ⚠️ Medium |
| **Automation** | Auto-assign rules | ✅ High |
| | Auto-label based on content | ⚠️ Medium |
| | SLA policies | ⚠️ Medium |
| **CSAT** | Customer satisfaction surveys | ✅ High |
| | CSAT reports and scores | ✅ High |
| **Reports** | Conversation reports (volume, resolution time) | ✅ High |
| | Agent performance reports | ✅ High |
| | Team reports | ⚠️ Medium |
| | Download reports (CSV) | ⚠️ Medium |
| **Knowledge Base** | Help center / articles | ⚠️ Medium |
| **Contacts** | Contact management and search | ✅ High |
| | Contact notes and attributes | ⚠️ Medium |
| | Contact segments | ⚠️ Medium |
| **Notifications** | In-app notifications | ✅ High |
| | Email notifications | ✅ High |
| | Push notifications (browser) | ⚠️ Medium |

---

## 9. Security

| Aspect | Implementation |
|---|---|
| HTTPS | Mandatory — Railway provides automatic SSL |
| Authentication | Email + password (Chatwoot built-in) |
| SECRET_KEY_BASE | Generated with `openssl rand -hex 64`, stored in Railway env vars |
| Database | Accessible only via Railway private network |
| Redis | Accessible only via Railway private network |
| WhatsApp tokens | Stored encrypted in Chatwoot database |
| File uploads | Authenticated access via signed URLs (Cloudflare R2) |
| Super Admin | Restricted access — only logicfy team |

---

## 10. Project Deliverables

### Phase 1 — Infrastructure Deployment (Days 1-4)

| # | Deliverable | Description |
|---|---|---|
| 1 | Railway Project Setup | Create Railway project with all 4 services (Web, Sidekiq, PostgreSQL, Redis) |
| 2 | Chatwoot Deployment | Deploy Chatwoot CE using versioned Docker image (`v3.10.x-ce`) |
| 3 | Database Configuration | PostgreSQL with pgvector extension, run initial migrations |
| 4 | Redis Configuration | Redis for caching and Sidekiq job queue |
| 5 | Custom Domain | Configure `chat.logicfy.co` with SSL |
| 6 | Cloudflare R2 Setup | Create bucket, configure CORS, set storage env vars |
| 7 | Resend Email Setup | Configure SMTP for both Web and Sidekiq services |
| 8 | Super Admin Account | Create the initial super admin account |

### Phase 2 — WhatsApp Cloud API Integration (Days 4-7)

| # | Deliverable | Description |
|---|---|---|
| 9 | Facebook App Creation | Create Facebook App with WhatsApp + Login for Business products |
| 10 | Embedded Signup Configuration | Configure the WhatsApp Embedded Signup flow in Meta Developer Portal |
| 11 | Chatwoot WhatsApp Config | Set WHATSAPP_APP_ID, WHATSAPP_CONFIGURATION_ID, WHATSAPP_APP_SECRET |
| 12 | End-to-End Test: Cloud API | Create test account → Embedded Signup → send/receive messages |
| 13 | 24h Window Test | Verify template selector appears when conversation window expires |
| 14 | Media Test | Send and receive: images, audio, video, documents via Cloud API |

### Phase 3 — Evolution API Integration (Days 7-9)

| # | Deliverable | Description |
|---|---|---|
| 15 | Evolution API Chatwoot Config | Enable CHATWOOT_ENABLED=true in Evolution API |
| 16 | Integration Test | Connect Evolution API instance to Chatwoot account → verify message sync |
| 17 | Bidirectional Test | Send from Chatwoot → appears on WhatsApp; receive on WhatsApp → appears in Chatwoot |
| 18 | Media Test (Baileys) | Send and receive all media types through Evolution API → Chatwoot |

### Phase 4 — Features & Validation (Days 9-12)

| # | Deliverable | Description |
|---|---|---|
| 19 | Multi-Account Test | Create 5+ accounts, each with 2+ agents, verify isolation |
| 20 | CSAT Configuration | Enable and test customer satisfaction surveys |
| 21 | Canned Responses | Set up and test quick reply templates |
| 22 | Automation Rules | Configure auto-assign and test with incoming conversations |
| 23 | Reports Validation | Verify conversation and agent reports generate correctly |
| 24 | Notification Test | Verify email notifications (new conversation, assignment) via Resend |
| 25 | Knowledge Base | Create a test help center article |
| 26 | Deployment Documentation | Complete setup guide to replicate the deployment from scratch |

---

## 11. Timeline

| Phase | Duration | Dates (Estimated) |
|---|---|---|
| Phase 1 — Infrastructure | 4 days | Days 1-4 |
| Phase 2 — Cloud API | 3 days | Days 4-7 |
| Phase 3 — Evolution API | 2 days | Days 7-9 |
| Phase 4 — Features & QA | 3 days | Days 9-12 |
| **Total** | **~12 working days** | **~2.5 weeks** |

> **Note:** Timeline assumes AI-assisted development. The Facebook App review process (required for production Embedded Signup) may take additional days and is outside the developer's control.

---

## 12. Risks and Mitigations

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Facebook App review takes longer than expected | High | Medium | Start the review process on Day 1; use test mode while waiting |
| pgvector not available on Railway PostgreSQL | Medium | High | Use Railway's pgvector template or install extension manually |
| Cloudflare R2 upload failures (checksum issue) | Medium | Medium | Pre-configure `storage.yml` with checksum workaround |
| Sidekiq memory growth with 500 accounts | Medium | Medium | Monitor memory, configure Sidekiq `max_retries` and concurrency |
| Railway costs higher than expected at scale | Low | Medium | Start with Phase 1 resources, scale incrementally based on actual usage |
| Evolution API Chatwoot sync loses messages | Low | High | Enable `CHATWOOT_MESSAGE_READ` and `CHATWOOT_BOT_CONTACT` for reliability |

---

## 13. Acceptance Criteria

The project is considered successful when:

1. ✅ Chatwoot is deployed on Railway and accessible at `https://chat.logicfy.co`
2. ✅ A user can connect their WhatsApp number via Embedded Signup (Cloud API) and send/receive messages
3. ✅ A user can connect their WhatsApp number via Evolution API (Baileys) and send/receive messages
4. ✅ Cloud API conversations correctly track the 24-hour window and show template selector when expired
5. ✅ All media types work (images, audio, video, documents) on both connection paths
6. ✅ Multiple accounts are isolated — agents can only see their own account's conversations
7. ✅ Agent invitations work via email (Resend SMTP)
8. ✅ CSAT surveys can be sent and results appear in reports
9. ✅ Canned responses and automation rules function correctly
10. ✅ Media files are stored in and served from Cloudflare R2
11. ✅ The service survives a redeploy without losing data or conversations

---

## 14. Dependencies and Requirements

### External Services

| Service | Action Required | Notes |
|---|---|---|
| **Railway** | Create project | 4 services: Web, Sidekiq, PostgreSQL, Redis |
| **Cloudflare R2** | Create bucket + API keys | `logicfy-chatwoot-media` bucket |
| **Resend** | Create account + verify domain | Verify `logicfy.co` domain for sending |
| **Meta Developer Portal** | Create Facebook App | WhatsApp + Login for Business products |
| **DNS Provider** | Add CNAME for `chat.logicfy.co` | Point to Railway's provided domain |

### Relationship with Evolution API Project

| Aspect | Details |
|---|---|
| Shared? | No — Chatwoot is a separate Railway project with its own database and Redis |
| Connection | Evolution API connects TO Chatwoot via Chatwoot's API (Account ID + Token) |
| Dependency | Evolution API must be deployed first (see `docs/PRD_evolution_api.md`) |
| Database | Completely separate — Chatwoot stores its own conversation history |

---

## 15. Open Questions / Research Tasks

| # | Question | Assigned to | Status |
|---|---|---|---|
| 1 | Does Railway's pgvector PostgreSQL template work with Chatwoot out of the box? | Samuel | Pending |
| 2 | How long does Facebook App review take for WhatsApp Embedded Signup? | Samuel + Jhon | Pending |
| 3 | Can the Chatwoot `storage.yml` be modified in the Docker image without rebuilding? (for R2 checksum fix) | Samuel | Pending |
| 4 | What is the monthly Railway cost for the 4-service Chatwoot stack at Phase 1 scale? | Samuel | Pending |
| 5 | Does Chatwoot support limiting the number of agents per account programmatically? | Samuel | Pending |
| 6 | Can Evolution API and Cloud API inboxes coexist in the same Chatwoot account? | Samuel | Pending |

---

## 16. Documentation Resources

| Resource | URL |
|---|---|
| Chatwoot GitHub | [github.com/chatwoot/chatwoot](https://github.com/chatwoot/chatwoot) |
| Chatwoot Self-Hosted Docs | [chatwoot.com/docs/self-hosted](https://www.chatwoot.com/docs/self-hosted) |
| Chatwoot Docker Guide | [chatwoot.com/docs/self-hosted/deployment/docker](https://www.chatwoot.com/docs/self-hosted/deployment/docker) |
| Chatwoot WhatsApp Setup | [chatwoot.com/docs/product/channels/whatsapp](https://www.chatwoot.com/docs/product/channels/whatsapp) |
| Chatwoot Platform API | [chatwoot.com/docs/product/others/platform-api](https://www.chatwoot.com/docs/product/others/platform-api) |
| Meta Developer Portal | [developers.facebook.com](https://developers.facebook.com/) |
| Meta WhatsApp Cloud API | [developers.facebook.com/docs/whatsapp/cloud-api](https://developers.facebook.com/docs/whatsapp/cloud-api) |
| Railway Docs | [docs.railway.com](https://docs.railway.com) |
| Evolution API PRD | `docs/PRD_evolution_api.md` in this repo |
| Wasender API Docs (reference) | `docs/wasender_api_docs.md` in this repo |
