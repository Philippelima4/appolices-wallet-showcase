# Architecture & Flows

This document describes the high-level system architecture and the two core product flows of Apólices Wallet.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        User / Browser                           │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│          Frontend — React 18 + Vite + Material UI               │
│          PWA · OAuth flows · Vercel · wallet.appolices.pt        │
└──────────────────────────┬──────────────────────────────────────┘
                           │ REST API
┌──────────────────────────▼──────────────────────────────────────┐
│          Backend — Strapi v5 (Node.js) · Render                 │
│                                                                 │
│   ┌─────────────────┐  ┌─────────────────┐  ┌───────────────┐  │
│   │  Email Scanner  │  │  OCR Pipeline   │  │ PDF Generator │  │
│   │  (IMAP batched) │  │  (GPT-4o Vision)│  │  (jsPDF)      │  │
│   └─────────────────┘  └─────────────────┘  └───────────────┘  │
└─────────┬──────────────────────┬──────────────────┬────────────┘
          │                      │                  │
┌─────────▼──────┐  ┌────────────▼──────┐  ┌───────▼──────────┐
│ Neon           │  │ Cloudinary        │  │ Resend           │
│ PostgreSQL     │  │ File Storage      │  │ Transactional    │
│ Policies, logs │  │ PDFs, documents   │  │ Email            │
└────────────────┘  └───────────────────┘  └──────────────────┘
          │                                          │
┌─────────▼──────────────┐   ┌────────────────────────────────┐
│ OpenAI GPT-4o Vision   │   │ Gmail / Outlook OAuth2         │
│ Document OCR           │   │ IMAP email scanning            │
└────────────────────────┘   └────────────────────────────────┘
```

| Layer | Technology |
|---|---|
| Frontend | React 18, Vite, Material UI, React Router |
| Backend | Strapi v5 (Node.js), custom controllers & services |
| Database | PostgreSQL via Neon (serverless) |
| File Storage | Cloudinary |
| Email | Resend (`strapi-provider-email-resend`) |
| AI / OCR | OpenAI GPT-4o Vision API |
| Auth | Gmail OAuth2, Outlook OAuth2, IMAP |
| PDF | jsPDF |
| Deployment | Vercel (frontend), Render (backend) |
| Mobile | PWA (Web App Manifest + Service Worker), Android APK |

---

## Flow 1 — AI-Powered Policy Import (OCR)

The platform automatically detects and imports insurance policies from the user's email inbox, using a multi-stage AI pipeline with a human-review step before any data is persisted.

```
User connects Gmail or Outlook
           │  OAuth2 consent flow
           ▼
Backend scans inbox via IMAP
           │  Batched processing · insurance keyword filter
           ▼
PDF attachments extracted
           │  Uploaded to Cloudinary
           ▼
GPT-4o Vision analyses each document
           │  Extracts structured policy fields
           ▼
Pending confirmation queue
           │  User reviews and corrects extracted data
           ▼
Policy saved to database
           │  Confirmation email sent via Resend
           ▼
           ✓  Policy visible in user's wallet
```

**Key design decisions:**
- IMAP scanning uses batched processing to stay within Render's free tier memory limits
- All OCR results enter a pending queue — no data is saved without explicit user confirmation
- The pipeline is fully asynchronous; users are notified by email when results are ready

---

## Flow 2 — Mediation Transfer

When a client transfers an insurance policy to the broker, the platform automates the generation and delivery of all legally required documentation, compliant with Portuguese insurance regulation.

```
Client selects policy to transfer
           │  In-app policy management
           ▼
Backend generates PDF transfer letter
           │  References: DL 72/2008, DL 109-D/2021, ASF Circular 3/2016
           ▼
PDF stored on Cloudinary
           │  Linked to the policy record
           ▼
        ┌──┴──────────────────────┐
        ▼                         ▼
Email to client             Email to insurer
PDF + transfer notice       Official mediation request
        └──────────┬──────────────┘
                   ▼
Policy status updated in database
           │  Transfer recorded · broker commission logged
           ▼
           ✓  Mediation complete
```

**Key design decisions:**
- All generated PDFs explicitly reference the applicable Portuguese insurance law articles
- Emails are sent simultaneously to both the client and the insurer in a single operation
- The full transfer history is preserved as an audit trail on the policy record

---

*For questions about this architecture, reach out via [LinkedIn](https://linkedin.com/in/philippe-limaa).*
