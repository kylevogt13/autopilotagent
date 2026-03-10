# DocuSign ISV Integration  -  Technical Spec

**Company:** EscrowEye
**Domain:** California real estate escrow  -  CAR form e-signatures
**Target:** Early March 2026
**API:** DocuSign eSign REST API v2.1

---

## 1. Architecture Overview

```
┌─────────────┐       ┌──────────────────┐       ┌─────────────┐
│  EscrowEye  │──────▶│  DocuSign eSign   │──────▶│  Signers    │
│  Backend    │◀──────│  REST API v2.1    │       │  (Buyers,   │
│             │webhook│                    │       │  Sellers,   │
│  - Auth     │       │  - Envelopes      │       │  Agents)    │
│  - Envelope │       │  - Templates      │       └─────────────┘
│  - Status   │       │  - Connect/Hooks  │
└─────────────┘       └──────────────────┘
```

EscrowEye sends envelopes via API, DocuSign delivers documents to signers, and Connect webhooks push status updates back to EscrowEye in real time.

---

## 2. Authentication

### 2.1 OAuth Grant Type: JWT (JSON Web Token)

JWT is the right choice for EscrowEye. Our integration operates server-to-server with a system account  -  no interactive user login required per transaction.

**Why JWT over Authorization Code:**
- Escrow workflows are automated  -  no user browser session available
- Single service account sends all envelopes on behalf of the company
- Unattended, long-running process needs token refresh without user interaction

### 2.2 Setup Requirements

| Item | Description |
|------|-------------|
| Integration Key (Client ID) | Created in DocuSign Apps and Keys admin |
| RSA Key Pair | Generated during app setup; private key stored securely |
| Service Account User ID | The DocuSign user GUID that envelopes are sent as |
| Consent | One-time admin consent grant for the service account |
| ISV Account | ISV 2.1 sandbox + production accounts from DocuSign partner program |

### 2.3 Token Flow

```
1. Build JWT assertion:
   - iss: Integration Key
   - sub: Service Account User ID
   - aud: account-d.docusign.com (demo) or account.docusign.com (prod)
   - iat: current timestamp
   - exp: iat + 3600 (max 1 hour)
   - scope: "signature impersonation"
   Sign with RSA private key (RS256)

2. POST https://account-d.docusign.com/oauth/token
   Body: grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&assertion={JWT}
   Response: { "access_token": "...", "token_type": "Bearer", "expires_in": 3600 }

3. GET https://account-d.docusign.com/oauth/userinfo
   Header: Authorization: Bearer {access_token}
   Response: { "accounts": [{ "account_id": "...", "base_uri": "https://na3.docusign.net" }] }

4. Cache base_uri + account_id. Refresh token before expiry (recommend at 50min mark).
```

### 2.4 Base URI Discovery

DocuSign hosts across multiple regions. After obtaining a token, call `/oauth/userinfo` to get the correct `base_uri` for API calls. Cache this  -  it rarely changes. All subsequent API calls use:

```
{base_uri}/restapi/v2.1/accounts/{account_id}/...
```

### 2.5 Secret Management

| Secret | Storage |
|--------|---------|
| RSA Private Key | Encrypted secrets manager (e.g., AWS Secrets Manager, Vault) |
| Integration Key | Environment variable |
| Service Account User ID | Environment variable |
| Access Token | In-memory cache with TTL, never persisted to disk |

---

## 3. Document Templates

### 3.1 CAR Form Template Strategy

Pre-create DocuSign templates for each CAR form type used in escrow transactions. Templates define document layout, signer roles, tab (field) positions, and routing order.

**Core CAR Forms to Template:**

| Form | Template Name | Signer Roles | Routing Order |
|------|--------------|-------------|---------------|
| RPA (Residential Purchase Agreement) | `ESC_RPA` | Buyer, Seller, Buyer Agent, Seller Agent | 1→2→3→4 |
| TDS (Transfer Disclosure Statement) | `ESC_TDS` | Seller, Buyer | 1→2 |
| AVID (Agent Visual Inspection Disclosure) | `ESC_AVID` | Agent | 1 |
| SPQ (Seller Property Questionnaire) | `ESC_SPQ` | Seller | 1 |
| AD (Agency Disclosure) | `ESC_AD` | Buyer, Seller, Buyer Agent, Seller Agent | 1→2→3→4 |
| Escrow Instructions | `ESC_INSTRUCTIONS` | Buyer, Seller | 1→2 |
| Amendment | `ESC_AMENDMENT` | Buyer, Seller | 1→2 |

### 3.2 Template Configuration

Each template uses:
- **Anchor tags** for tab placement  -  embed invisible text anchors in source PDFs (e.g., `//signer1_sign//`, `//signer1_date//`) so tabs auto-position regardless of document reformatting
- **Role-based routing**  -  signers assigned by role name (`Buyer`, `Seller`, etc.), mapped at envelope creation time to actual recipients
- **Merge fields**  -  property address, escrow number, transaction amount populated at send time via `textTabs` with `tabLabel` matching

### 3.3 Template Management

Templates are created once via DocuSign admin UI or API, then referenced by `templateId` in envelope creation. Store template IDs in a config table:

```sql
CREATE TABLE docusign_templates (
  id            UUID PRIMARY KEY,
  form_type     VARCHAR(50) NOT NULL UNIQUE,  -- e.g., 'RPA', 'TDS'
  template_id   VARCHAR(100) NOT NULL,         -- DocuSign template GUID
  template_name VARCHAR(200) NOT NULL,
  version       INT DEFAULT 1,
  active        BOOLEAN DEFAULT TRUE,
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 4. Envelope Creation

### 4.1 Create Envelope from Template

```
POST {base_uri}/restapi/v2.1/accounts/{account_id}/envelopes
Authorization: Bearer {access_token}
Content-Type: application/json
```

```json
{
  "templateId": "{template_id}",
  "templateRoles": [
    {
      "roleName": "Buyer",
      "name": "Jane Smith",
      "email": "jane@example.com",
      "routingOrder": "1",
      "tabs": {
        "textTabs": [
          { "tabLabel": "PropertyAddress", "value": "123 Main St, Los Angeles, CA 90001" },
          { "tabLabel": "EscrowNumber", "value": "ESC-2026-0042" },
          { "tabLabel": "PurchasePrice", "value": "$850,000" }
        ]
      }
    },
    {
      "roleName": "Seller",
      "name": "John Doe",
      "email": "john@example.com",
      "routingOrder": "2"
    }
  ],
  "status": "sent",
  "emailSubject": "EscrowEye  -  Documents for 123 Main St (ESC-2026-0042)",
  "emailBlurb": "Please review and sign the attached escrow documents."
}
```

**Response includes `envelopeId`**  -  store this for tracking.

### 4.2 Composite Templates (Multiple Documents)

For transactions requiring multiple CAR forms in a single signing session:

```json
{
  "compositeTemplates": [
    {
      "compositeTemplateId": "1",
      "serverTemplates": [
        { "sequence": "1", "templateId": "{RPA_template_id}" }
      ],
      "inlineTemplates": [
        {
          "sequence": "1",
          "recipients": {
            "signers": [
              { "roleName": "Buyer", "name": "Jane Smith", "email": "jane@example.com", "recipientId": "1" }
            ]
          }
        }
      ]
    },
    {
      "compositeTemplateId": "2",
      "serverTemplates": [
        { "sequence": "2", "templateId": "{TDS_template_id}" }
      ]
    }
  ],
  "status": "sent"
}
```

### 4.3 Envelope Tracking Table

```sql
CREATE TABLE docusign_envelopes (
  id              UUID PRIMARY KEY,
  transaction_id  UUID NOT NULL REFERENCES transactions(id),
  envelope_id     VARCHAR(100) NOT NULL UNIQUE,  -- DocuSign envelope GUID
  template_type   VARCHAR(50) NOT NULL,
  status          VARCHAR(30) NOT NULL DEFAULT 'sent',
  sent_at         TIMESTAMPTZ DEFAULT NOW(),
  completed_at    TIMESTAMPTZ,
  voided_at       TIMESTAMPTZ,
  metadata        JSONB,
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_envelopes_transaction ON docusign_envelopes(transaction_id);
CREATE INDEX idx_envelopes_status ON docusign_envelopes(status);
```

---

## 5. Webhook Handling  -  DocuSign Connect

### 5.1 Approach: Account-Level Connect Configuration

Use DocuSign Connect (not per-envelope `eventNotification`) because:
- Single configuration covers all envelopes from the account
- Survives envelope creation failures (no webhook config lost)
- Supports retry and logging at the platform level
- Required for ISV-grade reliability

### 5.2 Connect Configuration

Set up via DocuSign Admin > Connect:

| Setting | Value |
|---------|-------|
| URL | `https://api.escroweye.com/webhooks/docusign` |
| Events | `envelope-sent`, `envelope-delivered`, `envelope-completed`, `envelope-declined`, `envelope-voided`, `recipient-sent`, `recipient-delivered`, `recipient-completed`, `recipient-declined` |
| Format | JSON |
| Include Documents | No (fetch on demand to reduce payload size) |
| Include Certificate of Completion | No |
| Require Acknowledgement | Yes (200 response within 100s) |
| Retry | Enabled  -  DocuSign retries failed deliveries |
| HMAC Security | Enabled  -  verify `X-DocuSign-Signature-1` header |

### 5.3 Webhook Endpoint

```
POST /webhooks/docusign
Headers:
  X-DocuSign-Signature-1: {HMAC-SHA256 signature}
  Content-Type: application/json
```

### 5.4 HMAC Verification

DocuSign signs the payload with your HMAC secret key. Verify before processing:

```python
import hmac, hashlib, base64

def verify_docusign_hmac(payload: bytes, signature: str, hmac_key: str) -> bool:
    computed = base64.b64encode(
        hmac.new(hmac_key.encode(), payload, hashlib.sha256).digest()
    ).decode()
    return hmac.compare_digest(computed, signature)
```

**Reject requests that fail HMAC verification with 401.**

### 5.5 Webhook Payload Processing

Key fields in the Connect JSON payload:

```json
{
  "event": "envelope-completed",
  "apiVersion": "v2.1",
  "uri": "/restapi/v2.1/accounts/.../envelopes/{envelopeId}",
  "data": {
    "accountId": "...",
    "envelopeId": "abc-123-def",
    "envelopeSummary": {
      "status": "completed",
      "statusChangedDateTime": "2026-03-01T15:30:00Z",
      "recipients": {
        "signers": [
          {
            "name": "Jane Smith",
            "email": "jane@example.com",
            "status": "completed",
            "signedDateTime": "2026-03-01T15:28:00Z"
          }
        ]
      }
    }
  }
}
```

### 5.6 Status Mapping

| DocuSign Status | EscrowEye Action |
|----------------|-----------------|
| `sent` | Mark envelope as sent, notify escrow officer |
| `delivered` | Log  -  signer has viewed the documents |
| `completed` | Mark envelope complete, trigger document download, advance transaction workflow |
| `declined` | Alert escrow officer, flag transaction for review |
| `voided` | Mark envelope voided, log reason |

### 5.7 Webhook Processing Pipeline

```
1. Receive POST → verify HMAC → parse JSON
2. Idempotency check: deduplicate by envelopeId + event + statusChangedDateTime
3. Look up envelope in docusign_envelopes table
4. Update envelope status
5. If completed: enqueue async job to download signed documents via API
6. Trigger internal notifications (escrow officer dashboard, email alerts)
7. Return 200 OK within 100 seconds
```

### 5.8 Document Download (Post-Completion)

When an envelope completes, fetch the signed documents:

```
GET {base_uri}/restapi/v2.1/accounts/{account_id}/envelopes/{envelopeId}/documents/combined
Authorization: Bearer {access_token}
Accept: application/pdf
```

Store the signed PDF in your document management system linked to the transaction.

### 5.9 Webhook Event Log Table

```sql
CREATE TABLE docusign_webhook_events (
  id              UUID PRIMARY KEY,
  envelope_id     VARCHAR(100) NOT NULL,
  event_type      VARCHAR(50) NOT NULL,
  status          VARCHAR(30),
  payload         JSONB NOT NULL,
  processed_at    TIMESTAMPTZ,
  idempotency_key VARCHAR(200) NOT NULL UNIQUE,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_webhook_envelope ON docusign_webhook_events(envelope_id);
```

---

## 6. Error Handling

| Scenario | Response | Action |
|----------|----------|--------|
| 401 Unauthorized | Token expired | Refresh JWT token, retry once |
| 400 Bad Request | Invalid envelope data | Log error, alert developer, do not retry |
| 429 Rate Limited | Too many requests | Exponential backoff (1s, 2s, 4s, max 60s) |
| 5xx Server Error | DocuSign outage | Retry with backoff, alert if persistent (>3 failures) |
| Webhook HMAC failure | Invalid signature | Return 401, log for security review |
| Duplicate webhook | Already processed | Return 200, skip processing |

### Rate Limits

DocuSign ISV accounts have higher rate limits than standard accounts. Monitor `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers. Typical burst limit: 10 requests/second, sustained: ~1,000/hour for ISV.

---

## 7. Implementation Plan

### Phase 1  -  Foundation (Days 1–3)

- [ ] Register DocuSign ISV developer account and sandbox
- [ ] Create Integration Key (app) with JWT grant
- [ ] Generate RSA key pair, store private key in secrets manager
- [ ] Grant admin consent for service account
- [ ] Implement auth module: JWT assertion → token → base URI discovery
- [ ] Write integration tests against sandbox

### Phase 2  -  Templates (Days 4–6)

- [ ] Design anchor tag placement for top 3 CAR forms (RPA, TDS, Escrow Instructions)
- [ ] Create source PDFs with embedded anchor tags
- [ ] Upload templates to DocuSign sandbox via admin UI
- [ ] Store template IDs in config table
- [ ] Build template mapping service: form type → template ID → recipient roles

### Phase 3  -  Envelope Creation (Days 7–9)

- [ ] Implement envelope creation service (single template)
- [ ] Implement composite template support (multi-document)
- [ ] Build recipient mapping: EscrowEye transaction parties → DocuSign roles
- [ ] Add merge field population from transaction data
- [ ] Create docusign_envelopes tracking table
- [ ] Integration tests: send envelopes in sandbox, verify delivery

### Phase 4  -  Webhooks (Days 10–12)

- [ ] Set up Connect configuration in DocuSign sandbox
- [ ] Implement webhook endpoint with HMAC verification
- [ ] Build idempotent event processing pipeline
- [ ] Implement status update logic for each event type
- [ ] Add signed document download on envelope completion
- [ ] Create webhook event log table
- [ ] Test full lifecycle: send → sign → webhook → download

### Phase 5  -  Integration & Polish (Days 13–15)

- [ ] Wire envelope sending into EscrowEye transaction workflow UI
- [ ] Build escrow officer dashboard: envelope status per transaction
- [ ] Add email/notification alerts for key status changes
- [ ] Error handling: retry logic, dead letter queue for failed webhooks
- [ ] Rate limit handling with backoff

### Phase 6  -  Go-Live (Days 16–18)

- [ ] DocuSign ISV review and production app approval
- [ ] Production Connect configuration with HMAC keys
- [ ] Production secrets rotation and key management
- [ ] Deploy to production behind feature flag
- [ ] Smoke test with real transaction (internal test escrow)
- [ ] Remove feature flag, monitor for 48 hours
- [ ] Document runbook for operations team

---

## 8. Security Considerations

- **HMAC verification on all webhooks**  -  reject unsigned payloads
- **RSA private key** never in source control, only in secrets manager
- **Access tokens** in memory only, never logged or persisted
- **PII handling**  -  signer names/emails pass through DocuSign; minimize storage, encrypt at rest
- **Audit trail**  -  log all envelope operations for compliance (California DRE requirements)
- **IP allowlisting**  -  whitelist DocuSign Connect IP ranges for webhook endpoint (optional but recommended)

---

## 9. Monitoring & Observability

| Metric | Alert Threshold |
|--------|----------------|
| Envelope send failure rate | > 5% over 15 min |
| Webhook processing latency | > 10s p95 |
| Token refresh failure | Any failure |
| Webhook HMAC rejection rate | > 1% (possible misconfiguration or attack) |
| Envelope completion rate | < 80% after 48 hours (indicates signer drop-off) |

---

## References

- [DocuSign eSign REST API Reference](https://developers.docusign.com/docs/esign-rest-api/reference/)
- [Authentication Guide](https://developers.docusign.com/docs/esign-rest-api/esign101/auth/)
- [Templates Concepts](https://developers.docusign.com/docs/esign-rest-api/esign101/concepts/templates/)
- [Envelope Creation API](https://developers.docusign.com/docs/esign-rest-api/reference/envelopes/envelopes/create/)
- [DocuSign Connect (Webhooks)](https://www.docusign.com/blog/developers/connect-20)
- [Integration Planning Guide](https://www.docusign.com/blog/developers/docusign-esignature-integration-101-planning-your-integration)
