# DocuSign ISV Integration Spec  -  EscrowEye

> **Version:** 1.0
> **Date:** 2026-03-08
> **Author:** CEO Agent
> **Status:** Draft
> **Issue:** ESC-4

---

## 1. Overview

EscrowEye is a California real estate transaction SaaS. This spec defines the integration with DocuSign eSign REST API to handle electronic signing of California Association of Realtors (CAR) standard forms within the platform.

**Key forms:**
- **RPA**  -  Residential Purchase Agreement
- **TDS**  -  Transfer Disclosure Statement
- **SPQ**  -  Seller Property Questionnaire
- **AVID**  -  Agent Visual Inspection Disclosure

**Goals:**
1. Automate envelope creation for CAR forms tied to EscrowEye transactions.
2. Embedded signing  -  signers never leave the EscrowEye UI.
3. Real-time status sync via webhooks.
4. Audit trail stored locally for compliance.

**API Version:** DocuSign eSign REST API v2.1
**Base URLs:**
- Demo: `https://demo.docusign.net/restapi/v2.1`
- Production: `https://na1.docusign.net/restapi/v2.1` (varies by account region)

---

## 2. Authentication  -  OAuth 2.0 JWT Grant

EscrowEye operates as a backend service (no interactive user login per request), so we use the JWT Grant flow. This is the standard for ISV/server-to-server integrations.

### 2.1 Prerequisites

1. **DocuSign Developer Account** with an integration key (client ID) created at [DocuSign Admin Console](https://admindemo.docusign.com/).
2. **RSA Keypair**  -  generate a 2048-bit RSA key. Upload the public key to the integration key settings. Store the private key securely (e.g., AWS Secrets Manager or Vault).
3. **One-time Admin Consent**  -  the account admin must grant consent via this URL:

```
https://account-d.docusign.com/oauth/auth?
  response_type=code&
  scope=signature%20impersonation&
  client_id={INTEGRATION_KEY}&
  redirect_uri={REDIRECT_URI}
```

In production, use `account.docusign.com` instead of `account-d.docusign.com`.

### 2.2 JWT Token Request

**Endpoint:** `POST https://account-d.docusign.com/oauth/token`

**Request Body (form-encoded):**
```
grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&
assertion={JWT_TOKEN}
```

**JWT Claims:**

| Claim | Value |
|-------|-------|
| `iss` | Integration Key (Client ID) |
| `sub` | User ID of the impersonated user (GUID from DocuSign) |
| `aud` | `account-d.docusign.com` (demo) or `account.docusign.com` (prod) |
| `iat` | Current Unix timestamp |
| `exp` | `iat + 3600` (max 1 hour) |
| `scope` | `signature impersonation` |

**Sign the JWT** with RS256 using the private key.

**Response:**
```json
{
  "access_token": "eyJ0eXAi...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

### 2.3 Get User Info (Account ID + Base URI)

After obtaining the token, call:

```
GET https://account-d.docusign.com/oauth/userinfo
Authorization: Bearer {access_token}
```

**Response (relevant fields):**
```json
{
  "sub": "user-guid",
  "accounts": [
    {
      "account_id": "abc-123",
      "is_default": true,
      "base_uri": "https://demo.docusign.net"
    }
  ]
}
```

Use `base_uri` + `/restapi/v2.1/accounts/{account_id}` as the root for all subsequent API calls.

### 2.4 Token Caching Strategy

- Cache the access token in-memory (Redis or application cache).
- Refresh proactively at `expires_in - 300` seconds (5 min before expiry).
- On 401 response, force-refresh and retry once.

---

## 3. Envelope Creation

An envelope is DocuSign's container for documents + recipients + signing fields. Each EscrowEye transaction will create one composite envelope with all required CAR forms.

### 3.1 Create Envelope

**Endpoint:**
```
POST {base_uri}/restapi/v2.1/accounts/{account_id}/envelopes
Content-Type: application/json
Authorization: Bearer {access_token}
```

### 3.2 Request Body Structure

```json
{
  "emailSubject": "EscrowEye  -  Please sign documents for 123 Main St, Los Angeles, CA",
  "emailBlurb": "Documents for escrow #ESC-2026-0042 are ready for your signature.",
  "status": "sent",
  "documents": [
    {
      "documentId": "1",
      "name": "Residential Purchase Agreement (RPA)",
      "fileExtension": "pdf",
      "documentBase64": "<base64-encoded-pdf>"
    },
    {
      "documentId": "2",
      "name": "Transfer Disclosure Statement (TDS)",
      "fileExtension": "pdf",
      "documentBase64": "<base64-encoded-pdf>"
    },
    {
      "documentId": "3",
      "name": "Seller Property Questionnaire (SPQ)",
      "fileExtension": "pdf",
      "documentBase64": "<base64-encoded-pdf>"
    },
    {
      "documentId": "4",
      "name": "Agent Visual Inspection Disclosure (AVID)",
      "fileExtension": "pdf",
      "documentBase64": "<base64-encoded-pdf>"
    }
  ],
  "recipients": {
    "signers": [
      {
        "recipientId": "1",
        "routingOrder": "1",
        "name": "Jane Buyer",
        "email": "jane@example.com",
        "clientUserId": "buyer-001",
        "tabs": {
          "signHereTabs": [
            {
              "documentId": "1",
              "pageNumber": "12",
              "xPosition": "100",
              "yPosition": "700"
            }
          ],
          "dateSignedTabs": [
            {
              "documentId": "1",
              "pageNumber": "12",
              "xPosition": "300",
              "yPosition": "700"
            }
          ],
          "textTabs": [
            {
              "documentId": "1",
              "pageNumber": "1",
              "xPosition": "150",
              "yPosition": "200",
              "tabLabel": "BuyerName",
              "value": "Jane Buyer",
              "locked": "true"
            }
          ],
          "initialHereTabs": [
            {
              "documentId": "2",
              "pageNumber": "3",
              "xPosition": "100",
              "yPosition": "650"
            }
          ]
        }
      },
      {
        "recipientId": "2",
        "routingOrder": "1",
        "name": "John Seller",
        "email": "john@example.com",
        "clientUserId": "seller-001",
        "tabs": {
          "signHereTabs": [
            {
              "documentId": "1",
              "pageNumber": "12",
              "xPosition": "100",
              "yPosition": "750"
            }
          ],
          "dateSignedTabs": [
            {
              "documentId": "1",
              "pageNumber": "12",
              "xPosition": "300",
              "yPosition": "750"
            }
          ]
        }
      }
    ],
    "carbonCopies": [
      {
        "recipientId": "3",
        "routingOrder": "2",
        "name": "Lisa Agent",
        "email": "lisa@realty.com"
      }
    ]
  },
  "eventNotification": {
    "url": "https://api.escroweye.com/webhooks/docusign",
    "requireAcknowledgment": "true",
    "loggingEnabled": "true",
    "envelopeEvents": [
      { "envelopeEventStatusCode": "sent" },
      { "envelopeEventStatusCode": "delivered" },
      { "envelopeEventStatusCode": "completed" },
      { "envelopeEventStatusCode": "declined" },
      { "envelopeEventStatusCode": "voided" }
    ],
    "recipientEvents": [
      { "recipientEventStatusCode": "Sent" },
      { "recipientEventStatusCode": "Delivered" },
      { "recipientEventStatusCode": "Completed" },
      { "recipientEventStatusCode": "Declined" },
      { "recipientEventStatusCode": "AuthenticationFailed" }
    ]
  }
}
```

**Response (key fields):**
```json
{
  "envelopeId": "d4f3a1b2-...",
  "uri": "/envelopes/d4f3a1b2-...",
  "statusDateTime": "2026-03-08T12:00:00.000Z",
  "status": "sent"
}
```

### 3.3 Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| `clientUserId` on every signer | Enables embedded signing (signers are "captive recipients"). Without it, DocuSign sends email invites. |
| `routingOrder: "1"` for both buyer/seller | Parallel signing  -  both parties sign simultaneously, reducing transaction time. |
| `routingOrder: "2"` for carbon copies | Agents get notified after all signatures are collected. |
| `status: "sent"` | Immediately activates the envelope. Use `"created"` for draft envelopes. |
| Inline `eventNotification` | Per-envelope webhook config. Alternative: account-level Connect configuration. We use inline for granular control per transaction. |

### 3.4 Tab Types for CAR Forms

| Tab Type | Use Case |
|----------|----------|
| `signHereTabs` | Signature fields on each form |
| `initialHereTabs` | Initial fields (TDS and SPQ acknowledgments) |
| `dateSignedTabs` | Auto-populated date of signing |
| `textTabs` | Pre-filled fields (names, addresses, escrow numbers)  -  `locked: true` for read-only |
| `checkboxTabs` | Disclosure checkboxes on TDS/SPQ |
| `radioGroupTabs` | Yes/No selections on disclosure forms |

### 3.5 Tab Positioning  -  Anchor Strings (Recommended)

For CAR forms, use **anchor strings** instead of fixed x/y coordinates. Place invisible text markers in the PDF templates, and DocuSign positions tabs relative to them:

```json
{
  "signHereTabs": [
    {
      "anchorString": "/BuyerSignature/",
      "anchorXOffset": "20",
      "anchorYOffset": "-5",
      "anchorUnits": "pixels"
    }
  ]
}
```

**Advantages over fixed coordinates:**
- Survives minor PDF layout changes (page reflows, font size tweaks)
- Easier to maintain across form revisions
- Self-documenting  -  the anchor text describes what goes there

**Fallback:** Use fixed coordinates (`documentId` + `pageNumber` + `xPosition` + `yPosition`) when anchor strings are not feasible.

### 3.6 Using Templates (Alternative Approach)

For recurring form layouts, create templates in DocuSign with pre-placed tabs:

```
POST {base_uri}/restapi/v2.1/accounts/{account_id}/envelopes
```
```json
{
  "status": "sent",
  "compositeTemplates": [
    {
      "compositeTemplateId": "1",
      "serverTemplates": [
        {
          "sequence": "1",
          "templateId": "tmpl-rpa-v3"
        }
      ],
      "inlineTemplates": [
        {
          "sequence": "2",
          "recipients": {
            "signers": [
              {
                "recipientId": "1",
                "name": "Jane Buyer",
                "email": "jane@example.com",
                "clientUserId": "buyer-001",
                "roleName": "Buyer"
              }
            ]
          }
        }
      ]
    }
  ]
}
```

**Template strategy:** Create one template per CAR form type (RPA, TDS, SPQ, AVID) with standard tab placements. Use `compositeTemplates` to combine multiple templates into a single envelope. Override recipient details at send time via `inlineTemplates`.

---

## 4. Embedded Signing

Embedded signing keeps users inside the EscrowEye UI. The flow:

1. Create envelope with `clientUserId` on signers (see Section 3).
2. Generate a signing URL.
3. Render the URL in an iframe or redirect.

### 4.1 Create Recipient View (Signing URL)

**Endpoint:**
```
POST {base_uri}/restapi/v2.1/accounts/{account_id}/envelopes/{envelope_id}/views/recipient
Authorization: Bearer {access_token}
```

**Request Body:**
```json
{
  "returnUrl": "https://app.escroweye.com/transactions/{txn_id}/signing-complete?event=",
  "authenticationMethod": "none",
  "email": "jane@example.com",
  "userName": "Jane Buyer",
  "clientUserId": "buyer-001"
}
```

**Response:**
```json
{
  "url": "https://demo.docusign.net/Signing/MTRedeem/v1/..."
}
```

### 4.2 Return URL Events

DocuSign appends an event query parameter to `returnUrl`:

| Event | Meaning |
|-------|---------|
| `signing_complete` | Signer completed all required fields |
| `cancel` | Signer clicked "Finish Later" |
| `decline` | Signer declined to sign |
| `exception` | System error during signing |
| `session_timeout` | Signing session timed out |
| `ttl_expired` | URL expired before use (5 min default) |

### 4.3 Focused View (Recommended)

Instead of redirecting to the DocuSign URL, use the **docusign.js** library to render the signing ceremony inline within EscrowEye:

```html
<script src="https://js.docusign.com/bundle.js"></script>
<div id="docusign-signing"></div>

<script>
  window.DocuSign.loadDocuSign('{INTEGRATION_KEY}')
    .then((docusign) => {
      const signing = docusign.signing({
        url: signingUrl,  // from createRecipientView response
        displayFormat: 'focused',
        style: {
          branding: {
            primaryButton: {
              backgroundColor: '#1a73e8',
              color: '#ffffff'
            }
          }
        }
      });

      signing.on('sessionEnd', (event) => {
        // event.sessionEndType: 'signing_complete', 'decline', 'cancel', etc.
        handleSigningComplete(event.sessionEndType);
      });

      signing.mount('#docusign-signing');
    });
</script>
```

**Key advantages:** No redirect away from EscrowEye, minimalist UI that blends with branding, DOM events instead of URL redirects for completion handling.

**Requirements:** Set `frameAncestors` and `messageOrigins` in the `createRecipientView` request to include your app's origin:

```json
{
  "returnUrl": "https://app.escroweye.com/transactions/{txn_id}/signing-complete",
  "authenticationMethod": "none",
  "email": "jane@example.com",
  "userName": "Jane Buyer",
  "clientUserId": "buyer-001",
  "frameAncestors": ["https://app.escroweye.com"],
  "messageOrigins": ["https://app.escroweye.com"]
}
```

### 4.4 Implementation Notes

- The signing URL is single-use and expires after 5 minutes. Generate it on-demand when the user clicks "Sign Now."
- `email`, `userName`, and `clientUserId` must **exactly match** the values from the `createEnvelope` call. A mismatch produces `UNKNOWN_ENVELOPE_RECIPIENT`.
- After `signing_complete`, wait for the webhook confirmation before updating transaction status  -  the return URL event is client-side only.

---

## 5. Webhook Handling  -  DocuSign Connect

### 5.1 Configuration

We use **per-envelope inline webhooks** (configured via `eventNotification` in the envelope creation request, see Section 3.2). This avoids needing account-level Connect configuration.

For account-level fallback, configure via:
- DocuSign Admin Console > Connect > Add Configuration
- Or API: `POST {base_uri}/restapi/v2.1/accounts/{account_id}/connect`

### 5.2 Webhook Payload Structure

DocuSign sends XML by default. Request JSON with `"includeDocumentFields": "true"` in the event notification config.

**Example JSON payload (envelope completed):**
```json
{
  "event": "envelope-completed",
  "apiVersion": "v2.1",
  "uri": "/restapi/v2.1/accounts/{account_id}/envelopes/{envelope_id}",
  "retryCount": 0,
  "configurationId": 12345,
  "generatedDateTime": "2026-03-08T14:30:00.000Z",
  "data": {
    "accountId": "abc-123",
    "envelopeId": "d4f3a1b2-...",
    "userId": "user-guid",
    "envelopeSummary": {
      "status": "completed",
      "documentsUri": "/envelopes/d4f3a1b2-.../documents",
      "recipientsUri": "/envelopes/d4f3a1b2-.../recipients",
      "envelopeUri": "/envelopes/d4f3a1b2-...",
      "emailSubject": "EscrowEye  -  Please sign documents for 123 Main St",
      "sentDateTime": "2026-03-08T12:00:00.000Z",
      "completedDateTime": "2026-03-08T14:30:00.000Z",
      "recipients": {
        "signers": [
          {
            "name": "Jane Buyer",
            "email": "jane@example.com",
            "recipientId": "1",
            "status": "completed",
            "signedDateTime": "2026-03-08T13:00:00.000Z"
          },
          {
            "name": "John Seller",
            "email": "john@example.com",
            "recipientId": "2",
            "status": "completed",
            "signedDateTime": "2026-03-08T14:25:00.000Z"
          }
        ]
      }
    }
  }
}
```

### 5.3 HMAC Signature Verification

DocuSign signs webhook payloads with HMAC-SHA256. You configure an HMAC secret key in your Connect settings or inline config.

**Verification steps:**

1. Extract headers from the webhook request:
   - `X-DocuSign-Signature-1` (HMAC signature, base64-encoded)
2. Compute HMAC-SHA256 of the raw request body using your secret key.
3. Base64-encode the result.
4. Compare with the header value (constant-time comparison).

**Pseudocode:**
```python
import hmac, hashlib, base64

def verify_docusign_hmac(payload_bytes: bytes, secret: str, signature_header: str) -> bool:
    computed = base64.b64encode(
        hmac.new(
            secret.encode('utf-8'),
            payload_bytes,
            hashlib.sha256
        ).digest()
    ).decode('utf-8')
    return hmac.compare_digest(computed, signature_header)
```

### 5.4 Webhook Processing Logic

```
on webhook received:
  1. Verify HMAC signature → reject if invalid (401)
  2. Parse envelope_id from payload
  3. Look up transaction by envelope_id in docusign_envelopes table
  4. If not found → log warning, return 200 (avoid retries for unknown envelopes)
  5. Deduplicate: check if this event was already processed (idempotency key = envelope_id + event + timestamp)
  6. Update envelope status in docusign_envelopes table
  7. Update per-recipient status in docusign_recipients table
  8. If event = "envelope-completed":
     a. Download signed documents via GET .../envelopes/{id}/documents/combined
     b. Store in document storage (S3 or equivalent)
     c. Update EscrowEye transaction status
     d. Notify relevant parties via app notifications
  9. Return 200 OK within 100 seconds (DocuSign timeout)
```

### 5.5 Retry Behavior

DocuSign retries failed webhook deliveries:
- Up to 10 retries over 8 days
- Exponential backoff: 1h, 2h, 4h, 8h, 24h, 24h, 24h, 24h, 24h, 24h
- Always return 200 on success; 4xx/5xx triggers retry
- Implement idempotent processing to handle duplicate deliveries

---

## 6. Database Schema

Three tables to track DocuSign state alongside EscrowEye transactions.

### 6.1 `docusign_envelopes`

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID (PK) | Internal primary key |
| `transaction_id` | UUID (FK → transactions) | EscrowEye transaction reference |
| `envelope_id` | VARCHAR(36) UNIQUE NOT NULL | DocuSign envelope GUID |
| `status` | VARCHAR(20) NOT NULL | `created`, `sent`, `delivered`, `completed`, `declined`, `voided` |
| `email_subject` | TEXT | Envelope subject line |
| `documents_json` | JSONB | Metadata about included documents (form types, page counts) |
| `sent_at` | TIMESTAMP | When envelope was sent to recipients |
| `completed_at` | TIMESTAMP | When all signatures collected |
| `voided_at` | TIMESTAMP | If envelope was voided |
| `void_reason` | TEXT | Reason for voiding |
| `signed_documents_url` | TEXT | Storage URL for completed signed PDF bundle |
| `raw_webhook_payload` | JSONB | Last webhook payload (for debugging) |
| `created_at` | TIMESTAMP NOT NULL DEFAULT NOW() | |
| `updated_at` | TIMESTAMP NOT NULL DEFAULT NOW() | |

**Indexes:**
- `idx_envelopes_envelope_id` on `envelope_id` (unique, used for webhook lookups)
- `idx_envelopes_transaction_id` on `transaction_id` (FK join)
- `idx_envelopes_status` on `status` (filtering)

### 6.2 `docusign_recipients`

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID (PK) | Internal primary key |
| `envelope_id` | UUID (FK → docusign_envelopes.id) | Parent envelope |
| `recipient_id` | VARCHAR(10) NOT NULL | DocuSign recipient ID within envelope |
| `client_user_id` | VARCHAR(100) | For embedded signing correlation |
| `role` | VARCHAR(20) NOT NULL | `buyer`, `seller`, `agent`, `escrow_officer` |
| `name` | VARCHAR(255) NOT NULL | Recipient full name |
| `email` | VARCHAR(255) NOT NULL | Recipient email |
| `routing_order` | INTEGER NOT NULL DEFAULT 1 | Signing order |
| `status` | VARCHAR(20) NOT NULL | `created`, `sent`, `delivered`, `completed`, `declined`, `authentication_failed` |
| `signed_at` | TIMESTAMP | When this recipient signed |
| `declined_at` | TIMESTAMP | If recipient declined |
| `decline_reason` | TEXT | Reason for declining |
| `created_at` | TIMESTAMP NOT NULL DEFAULT NOW() | |
| `updated_at` | TIMESTAMP NOT NULL DEFAULT NOW() | |

**Indexes:**
- `idx_recipients_envelope_id` on `envelope_id`
- `idx_recipients_client_user_id` on `client_user_id`
- `idx_recipients_status` on `status`

### 6.3 `docusign_webhook_events`

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID (PK) | Internal primary key |
| `envelope_id` | UUID (FK → docusign_envelopes.id) | Related envelope |
| `event_type` | VARCHAR(50) NOT NULL | `envelope-sent`, `envelope-completed`, `recipient-completed`, etc. |
| `idempotency_key` | VARCHAR(255) UNIQUE NOT NULL | `{ds_envelope_id}:{event}:{generated_datetime}` |
| `payload` | JSONB NOT NULL | Full webhook payload |
| `processed` | BOOLEAN NOT NULL DEFAULT FALSE | Whether this event has been handled |
| `processed_at` | TIMESTAMP | When processing completed |
| `error` | TEXT | Error message if processing failed |
| `created_at` | TIMESTAMP NOT NULL DEFAULT NOW() | |

**Indexes:**
- `idx_webhook_events_idempotency` on `idempotency_key` (unique, dedup)
- `idx_webhook_events_envelope_id` on `envelope_id`
- `idx_webhook_events_processed` on `processed` WHERE `processed = FALSE` (partial index for retry queue)

### 6.4 Migration SQL

```sql
CREATE TABLE docusign_envelopes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id UUID NOT NULL REFERENCES transactions(id),
    envelope_id VARCHAR(36) UNIQUE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'created',
    email_subject TEXT,
    documents_json JSONB,
    sent_at TIMESTAMP,
    completed_at TIMESTAMP,
    voided_at TIMESTAMP,
    void_reason TEXT,
    signed_documents_url TEXT,
    raw_webhook_payload JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE docusign_recipients (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    envelope_id UUID NOT NULL REFERENCES docusign_envelopes(id) ON DELETE CASCADE,
    recipient_id VARCHAR(10) NOT NULL,
    client_user_id VARCHAR(100),
    role VARCHAR(20) NOT NULL,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    routing_order INTEGER NOT NULL DEFAULT 1,
    status VARCHAR(20) NOT NULL DEFAULT 'created',
    signed_at TIMESTAMP,
    declined_at TIMESTAMP,
    decline_reason TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE(envelope_id, recipient_id)
);

CREATE TABLE docusign_webhook_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    envelope_id UUID NOT NULL REFERENCES docusign_envelopes(id) ON DELETE CASCADE,
    event_type VARCHAR(50) NOT NULL,
    idempotency_key VARCHAR(255) UNIQUE NOT NULL,
    payload JSONB NOT NULL,
    processed BOOLEAN NOT NULL DEFAULT FALSE,
    processed_at TIMESTAMP,
    error TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_envelopes_envelope_id ON docusign_envelopes(envelope_id);
CREATE INDEX idx_envelopes_transaction_id ON docusign_envelopes(transaction_id);
CREATE INDEX idx_envelopes_status ON docusign_envelopes(status);
CREATE INDEX idx_recipients_envelope_id ON docusign_recipients(envelope_id);
CREATE INDEX idx_recipients_client_user_id ON docusign_recipients(client_user_id);
CREATE INDEX idx_recipients_status ON docusign_recipients(status);
CREATE INDEX idx_webhook_events_idempotency ON docusign_webhook_events(idempotency_key);
CREATE INDEX idx_webhook_events_envelope_id ON docusign_webhook_events(envelope_id);
CREATE INDEX idx_webhook_events_unprocessed ON docusign_webhook_events(processed) WHERE processed = FALSE;
```

---

## 7. Implementation Plan  -  18 Days

### Phase 1: Foundation (Days 1–6)

| Day | Task | Deliverable |
|-----|------|-------------|
| 1 | DocuSign developer account setup, integration key creation, RSA keypair generation | Credentials stored in secrets manager |
| 2 | Implement JWT auth module: token generation, caching, refresh logic | `DocuSignAuthService` with tests |
| 3 | Implement `/oauth/userinfo` call, account/base-URI resolution, config management | Auth module complete, end-to-end token flow verified in demo |
| 4 | Database migration: create all 3 tables | Migration scripts, rollback scripts |
| 5 | Envelope creation service: build request body from transaction data, map CAR form types to documents | `EnvelopeService.create()` with unit tests |
| 6 | Integration test: create envelope in DocuSign demo sandbox, verify it appears in DocuSign UI | End-to-end envelope creation working |

### Phase 2: Signing & Webhooks (Days 7–12)

| Day | Task | Deliverable |
|-----|------|-------------|
| 7 | Embedded signing: recipient view URL generation, return URL handling | `SigningService.getSigningUrl()` |
| 8 | Frontend: signing modal/iframe, handle return URL events, error states | Signing UI component |
| 9 | Webhook endpoint: receive, HMAC verify, parse, store in `docusign_webhook_events` | `POST /webhooks/docusign` endpoint |
| 10 | Webhook processor: update envelope/recipient status, idempotency checks | Event processing pipeline |
| 11 | Completed envelope handling: download signed PDFs, store in document storage, update transaction | Document retrieval + storage |
| 12 | Integration test: full flow  -  create envelope → sign (demo) → webhook → status update → PDF download | End-to-end flow verified in sandbox |

### Phase 3: Hardening & Launch (Days 13–18)

| Day | Task | Deliverable |
|-----|------|-------------|
| 13 | Template setup: create DocuSign templates for RPA, TDS, SPQ, AVID with standard tab placements | 4 templates configured in DocuSign |
| 14 | Composite template integration: refactor envelope creation to use server templates with inline overrides | Template-based envelope creation |
| 15 | Error handling: retry logic for API failures, webhook processing failures, dead letter queue for unprocessable events | Resilience layer |
| 16 | Void/resend support: API endpoints for voiding envelopes and resending to recipients | Envelope management features |
| 17 | Monitoring + alerting: webhook processing lag, failed deliveries, signing completion rates, API error rates | Dashboards and alerts |
| 18 | Production go-live: switch to prod credentials, DNS/firewall for webhook URL, smoke test, enable for first customer cohort | Production deployment |

### Key Milestones

- **Day 6:** Envelope creation working in sandbox
- **Day 12:** Full signing flow working end-to-end in sandbox
- **Day 18:** Production deployment

### Risk Mitigations

| Risk | Mitigation |
|------|------------|
| DocuSign rate limits (1000 requests/hour default for ISV) | Implement request queuing + exponential backoff; request higher limits from DocuSign ISV team |
| CAR form PDF layout changes | Use template-based tab placement; version templates per form revision |
| Webhook delivery failures | Idempotent processing + dead letter queue + manual retry UI |
| RSA key rotation | Support dual-key config; rotate without downtime |
| DocuSign sandbox/prod parity issues | Run integration tests against sandbox before every prod release |

---

## 8. Security Considerations

1. **RSA Private Key**  -  stored in secrets manager, never in source control, rotated annually.
2. **HMAC Secret**  -  separate secret for webhook verification, also in secrets manager.
3. **Access Tokens**  -  cached in-memory only, never persisted to disk or database.
4. **Signed PDFs**  -  encrypted at rest in document storage; access logged.
5. **PII in Logs**  -  redact signer emails and names from application logs; envelope IDs are safe to log.
6. **Webhook Endpoint**  -  HMAC verification required; rate limit inbound requests; HTTPS only.
7. **CSP Headers**  -  if using iframe for embedded signing, allowlist DocuSign domains in Content-Security-Policy frame-src.

---

## 9. API Error Handling

| HTTP Status | Meaning | Action |
|-------------|---------|--------|
| 400 | Bad request / validation error | Log, fix request, do not retry |
| 401 | Token expired or invalid | Refresh token, retry once |
| 403 | Insufficient permissions | Check scopes and user impersonation config |
| 404 | Envelope/recipient not found | Log, investigate |
| 429 | Rate limit exceeded | Back off per `Retry-After` header |
| 500+ | DocuSign server error | Retry with exponential backoff (max 3 retries) |

---

## Appendix A: Environment Variables

| Variable | Description |
|----------|-------------|
| `DOCUSIGN_INTEGRATION_KEY` | OAuth client ID |
| `DOCUSIGN_USER_ID` | GUID of impersonated user |
| `DOCUSIGN_ACCOUNT_ID` | DocuSign account ID |
| `DOCUSIGN_RSA_PRIVATE_KEY` | PEM-encoded private key (or path to secrets manager) |
| `DOCUSIGN_HMAC_SECRET` | HMAC key for webhook verification |
| `DOCUSIGN_BASE_URI` | `https://demo.docusign.net` or production equivalent |
| `DOCUSIGN_OAUTH_HOST` | `account-d.docusign.com` (demo) or `account.docusign.com` (prod) |

## Appendix B: CAR Form → Template Mapping

| Form | Template Name | Signers | Key Tabs |
|------|---------------|---------|----------|
| RPA | `escroweye-rpa-v1` | Buyer, Seller | Signatures (p12, p13), initials (p1-p11), purchase price (text), address (text), dates (date) |
| TDS | `escroweye-tds-v1` | Seller | Signatures (p5), checkboxes (p1-p4 disclosures), text fields (known defects) |
| SPQ | `escroweye-spq-v1` | Seller | Signatures (p4), radio groups (yes/no per question), text fields (explanations) |
| AVID | `escroweye-avid-v1` | Listing Agent, Selling Agent | Signatures (p2), text fields (inspection notes), date fields |
