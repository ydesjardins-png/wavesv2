# Backend API Contract — `apply.html`

This document specifies the API surface that `apply.html` expects from a
backend. Use it as the spec when building / commissioning the new backend.

> All endpoints originally lived at `https://web-production-31ce.up.railway.app`
> (now deprecated). A new backend just needs to expose the same surface at
> a single base URL (referred to as `{BACKEND_URL}` below) and `apply.html`
> will work without code changes — set `BACKEND_URL` and `IS_SANDBOX_FORM = false`
> at the top of `apply.html`.

CORS: backend must allow `https://wavesfinancial.ca` (and Netlify preview
domains if used).

---

## Endpoints

### 1. `POST {BACKEND_URL}/api/apply/new`

Submit a new loan application.

**Request body** (JSON):
```jsonc
{
  "applicantInfo": {
    "firstName": "string",
    "lastName":  "string",
    "email":     "string",
    "phone":     "string (E.164 or 10-digit CA)",
    "dob":       "YYYY-MM-DD",
    "sin":       "string (9 digits, no spaces) — optional",
    "address":   "string",
    "city":      "string",
    "province":  "BC|ON|NS|NB|PE|NL",
    "postal":    "A1A 1A1"
  },
  "loanInfo": {
    "amount":         500-1000,
    "purpose":        "string",
    "employmentType": "string",
    "employer":       "string",
    "monthlyIncome":  number,
    "payFrequency":   "weekly|biweekly|semimonthly|monthly"
  },
  "banking": {
    "flinksLoginId":     "string",
    "institution":       "string",
    "sandbox":           boolean,
    "transitNumber":     "string (5 digits)",
    "institutionNumber": "string (3 digits)",
    "accountNumber":     "string"
  },
  "consents": {
    "padAuthorization": true,
    "creditCheck":      true,
    "termsAccepted":    true
  },
  "ref": "WAV-XXXXXX (client-generated reference)"
}
```

**Response** (JSON):
```jsonc
{
  "ref":      "WAV-XXXXXX",
  "decision": "approved | manual_review | declined",
  "tier":     "gold | green | blue | yellow | orange | red",
  "score":    0-100,
  "approvedAmount":    number | null,
  "feasibleAmount":    number | null,
  "declineReason":     "string | null"
}
```

The frontend renders one of three outcome screens based on `decision`.

---

### 2. `POST {BACKEND_URL}/api/apply/renewal`

Submit a renewal application. Same shape as `/api/apply/new` plus:
```jsonc
{
  "renewalInfo": {
    "previousLoanRef": "WAV-XXXXXX",
    "requestedAmount": 500-1000
  }
}
```
Same response shape.

---

### 3. `GET {BACKEND_URL}/api/config/loan-settings`

Returns global loan amount/APR config. Used to populate slider bounds and
disclosure text.

**Response:**
```jsonc
{
  "apr":             0.23,           // decimal e.g. 0.23 = 23%
  "termDays":        112,
  "paymentCount":    8,
  "minLoan":         500,
  "maxLoan":         1000,
  "padCutoffTime":   "14:30",        // 24h, EST
  "servedProvinces": ["BC","ON","NS","NB","PE","NL"]
}
```

If the request fails, the frontend uses these defaults: APR 23%, 112 days, 8
payments, $500–$1,000, cutoff 2:30 PM EST.

---

### 4. `GET {BACKEND_URL}/api/config/optional-fees`

Returns the list of optional fees the applicant can opt into.

**Response:**
```jsonc
{
  "fees": [
    {
      "id":          "string",
      "label":       "string",
      "description": "string",
      "amount":      number,
      "default":     boolean
    }
  ]
}
```

If the request fails, the form renders with no optional fees (safe default).

---

### 5. `GET {BACKEND_URL}/api/config/payment-methods`

Returns the list of accepted payment methods (e.g. e-Transfer, Direct
Deposit) for funding.

**Response:**
```jsonc
{
  "methods": [
    {
      "id":          "string",
      "label":       "string",
      "description": "string",
      "default":     boolean
    }
  ]
}
```

If the request fails, the form falls back to the default radio options
already in the HTML.

---

### 6. `GET {BACKEND_URL}/api/ibv/embed-url`

Returns the iframe URL for instant bank verification (Flinks or VoPay iQ11).

**Response:**
```jsonc
{
  "provider":  "flinks | vopay",
  "flinksUrl": "https://...",   // if provider === 'flinks'
  "embedUrl":  "https://..."    // if provider === 'vopay'
}
```

If the request fails or returns no provider, the form falls back to
sandbox-mode bank picker.

---

## Decisioning Tiers (reference)

For parity with the old Railway backend's scoring engine:

| Tier   | Score  | Decision      |
|--------|--------|---------------|
| Gold   | 0–15   | Auto-approved |
| Green  | 15–30  | Manual review |
| Blue   | 30–50  | Manual review |
| Yellow | 50–70  | Manual review |
| Orange | 70–90  | Manual review (surface feasible amount) |
| Red    | 90–100 | Auto-declined |

Scoring inputs (from Flinks data + form): NSF events (≤25), payment
oppositions (≤20), DTI ratio with 75% cap (≤20), income regularity (≤15),
income mismatch (≤10), balance cushion (≤10).

---

## Quick smoke test

Once the new backend is live and `BACKEND_URL` is set:

```bash
# health check (if backend exposes one)
curl -i {BACKEND_URL}/health

# config fetch
curl -i {BACKEND_URL}/api/config/loan-settings

# submission (replace with full payload)
curl -X POST {BACKEND_URL}/api/apply/new \
  -H 'Content-Type: application/json' \
  -d '{"applicantInfo":{...},"loanInfo":{...},"banking":{...},"consents":{...},"ref":"WAV-TEST01"}'
```

Then in `apply.html` open the browser console — you should see no warnings
from `[loanSettings]`, `[OptionalFees]`, `[PaymentMethods]`, or `[IBV]`.
