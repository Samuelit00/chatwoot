AGENTS.md

---

## Logicfy Project Context

This section is specific to the Logicfy deployment of Chatwoot. Read this in addition to all guidelines above.

### What we're building

A self-hosted Chatwoot instance as the **human oversight panel** for Logicfy's WhatsApp AI agent SaaS product (logicfy.co). Logicfy's customers are businesses that use AI agents to handle WhatsApp support 24/7. Chatwoot provides human-in-the-loop supervision and escalation.

Target: multi-tenant, up to 500 business accounts × 10 agents each.

### Infrastructure stack

- **Hosting:** Railway (two services: `web` + `worker`)
- **DB:** PostgreSQL on Railway + `pgvector` extension (enable from day one)
- **Queue/cache:** Redis on Railway
- **Storage:** Cloudflare R2 (S3-compatible, zero egress cost) — never Railway's ephemeral disk
- **Email:** Resend via SMTP
- **Domain:** `chat.logicfy.co` → Cloudflare DNS (CNAME to Railway, proxy OFF, grey cloud)
- **WhatsApp:** Meta Graph API official — never Evolution API or Baileys

### Critical environment variables for this deployment

```
FRONTEND_URL=https://chat.logicfy.co
FORCE_SSL=true
ACTIVE_STORAGE_SERVICE=s3_compatible
STORAGE_BUCKET_NAME=<bucket>
STORAGE_ACCESS_KEY_ID=<r2-key>
STORAGE_SECRET_ACCESS_KEY=<r2-secret>
STORAGE_REGION=auto
STORAGE_ENDPOINT=https://<account-id>.r2.cloudflarestorage.com
STORAGE_FORCE_PATH_STYLE=true
MAILER_SENDER_EMAIL=noreply@logicfy.co
SMTP_ADDRESS=smtp.resend.com
SMTP_USERNAME=resend
SMTP_PASSWORD=<resend-api-key>
SMTP_PORT=465
ENABLE_ACCOUNT_SIGNUP=false
```

`DATABASE_URL` and `REDIS_URL` are injected automatically by Railway.

### Rules specific to this project

- Never commit secrets. `.env` must stay in `.gitignore`. Only `.env.example` with placeholders goes to git.
- Do not modify Chatwoot core source files unless strictly necessary. Prefer ENV vars, feature flags, and the `enterprise/` overlay pattern.
- All comments, commit messages, and documentation in **Spanish**.
- Before any Railway deploy, confirm with the user. Do not run `railway up` or push to production branch autonomously.
- When in doubt about a Chatwoot config option, check https://www.chatwoot.com/docs/self-hosted/configuration/environment-variables before modifying code.

### Deployment checklist state

- [x] WhatsApp Business account received from Logicfy
- [ ] Railway project created with PostgreSQL + Redis
- [ ] First deploy to Railway successful
- [ ] chat.logicfy.co domain + HTTPS working
- [x] Cloudflare R2 connected and persistence verified
- [x] Resend connected, invitation emails working
- [x] WhatsApp inbox created in Chatwoot
- [ ] Meta webhook pointing to chat.logicfy.co/webhooks/whatsapp
- [x] Outbound webhooks to Logicfy backend configured
- [ ] Postman collection exported and documented