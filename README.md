# Scaramouch1 - Persona API Proxy

A Node.js API proxy and webhook handler for [Persona](https://withpersona.com/), the identity verification platform. Handles identity verification sessions, stores verification state in Supabase, and forwards Persona webhooks with signature validation.

---

## Overview

Scaramouch1 is a server-to-server proxy that sits between a browser client (a Chrome extension or web app) and the Persona identity API. It:

1. Creates Persona inquiries - when a user needs to verify their identity, the proxy creates an inquiry on Persona and returns the verification URL.
2. Receives Persona webhooks - Persona POSTs verification events (created, completed, failed) to this server, which validates the HMAC signature and updates Supabase.
3. Tracks verification history - every inquiry is stored in a `verifications` table in Supabase with full webhook data.
4. Enforces CORS - only approved origins (Outlier app, Altropa, and one specific Chrome extension) can call the API.

---

## Features

- **Persona inquiry creation** - Server-side proxy to Persona's `/api/v1/inquiries` endpoint.
- **HMAC webhook verification** - Validates `Persona-Signature` header using `WEBHOOK_SECRET`.
- **Supabase persistence** - Stores every verification in the `verifications` table.
- **Session token auth** - API requests require a valid session token via `x-session-token`.
- **Allowlist CORS** - Only Outlier, Altropa, and one approved extension ID can make requests.
- **Usage logging** - Every request is appended to `usage-logs.json` for auditing.
- **Retry tracking** - Failed verifications are limited to 3 attempts before lockout.
- **AST-based deobfuscation utility** - Includes a Babel-based deobfuscator for stripping hex/unicode escapes, constant folding, and member expression normalization (used for sanitizing webhook payloads).

---

## Installation

### Prerequisites

- Node.js 18+
- A Supabase project
- A Persona account with a template created
- A webhook endpoint configured in Persona

### Setup

```bash
git clone https://github.com/muyaallan/Scaramouch1.git
cd Scaramouch1
npm install
```

### Environment variables

Create a `.env` file:

```env
PERSONA_API_KEY=persona_sandbox_xxxxxxxxxxxx
TEMPLATE_ID=itmpl_xxxxxxxxxxxxxxxx
WEBHOOK_SECRET=whsec_xxxxxxxxxxxxxxxx
SUPABASE_URL=https://xxxxx.supabase.co
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOi...
PORT=3000
```

### Database

Load the schema into your Supabase SQL editor. Open `database.sql` and run it.

### Run

```bash
npm start
```

The server listens on `http://localhost:3000`.

---

## API

### POST /api/inquiry

Create a new Persona verification inquiry.

**Headers**
```
x-session-token: <session-token>
```

**Body**
```json
{
  "referenceId": "user_abc123",
  "redirectUri": "https://app.outlier.ai/verify/callback"
}
```

**Response**
```json
{
  "inquiryId": "inq_abc123",
  "verificationUrl": "https://withpersona.com/verify?inquiry-id=inq_abc123"
}
```

### POST /api/webhook

Receives Persona webhook events. Validates the `Persona-Signature` header against `WEBHOOK_SECRET` before processing.

### GET /api/verification/:inquiryId

Retrieves the current verification state for an inquiry.

**Response**
```json
{
  "inquiryId": "inq_abc123",
  "status": "completed",
  "verificationStatus": "passed",
  "personaAccountId": "act_abc123",
  "remainingAttempts": 3,
  "failureReasons": []
}
```

---

## Project Structure

```text
Scaramouch1/
├── server.js                    # Express server with Persona proxy routes
├── deobfuscator.js              # Babel AST utility for sanitizing payloads
├── database.sql                 # Supabase schema for verifications table
├── package.json
├── response.txt                 # Sample Persona webhook payloads (for testing)
└── js-deobfuscator/             # Standalone AST deobfuscator (optional helper)
    ├── package.json
    └── server.js
```

---

## Database schema

The `verifications` table tracks every inquiry:

| Column | Type | Purpose |
| --- | --- | --- |
| `id` | bigint | Primary key |
| `inquiry_id` | text | Persona inquiry ID (unique) |
| `reference_id` | text | Your internal user reference |
| `user_id` | text | The user who triggered verification |
| `status` | text | Inquiry status (created, completed, failed) |
| `verification_status` | text | Result (passed, failed, requires-review) |
| `webhook_data` | jsonb | Full webhook payload |
| `persona_account_id` | text | Persona account ID |
| `template_id` | text | Persona template used |
| `redirect_uri` | text | Where to send the user after verification |
| `failure_reasons` | jsonb | Historical failure reasons |
| `latest_failure_reasons` | jsonb | Most recent failure reasons |
| `remaining_attempts` | integer | Attempts left (defaults to 3) |
| `created_at` | timestamptz | When the inquiry was created |
| `updated_at` | timestamptz | Last update timestamp |

Indexes on `inquiry_id`, `user_id`, `template_id`, `persona_account_id`, and `created_at` keep lookups fast.

---

## Security Notes

- **CORS allowlist** - Only `app.outlier.ai`, `altropa.onrender.com`, and one specific Chrome extension can make requests.
- **Session tokens** - All API requests must include a valid `x-session-token` header. Tokens are validated against Supabase.
- **Webhook signatures** - Every webhook must pass HMAC verification or it's rejected with 401.
- **Service role key** - The Supabase service role key bypasses RLS. It's stored only in the server env, never exposed to the client.
- **Usage logs** - Written to `usage-logs.json` (gitignored). Should be rotated or moved to Supabase for production.

---

## Deobfuscator utility

`deobfuscator.js` is a Babel AST utility that sanitizes obfuscated webhook payloads before storage:

1. **String literal normalization** - Decodes `\x6c\x6f\x67` back to `log`.
2. **Constant folding** - Evaluates `"hello" + " " + "world"` to `"hello world"` and `0x12 + 0x34` to `70`.
3. **Member expression normalization** - Converts `console["log"]` to `console.log`.

---

## Testing

`response.txt` contains two sample Persona inquiry webhooks - one for a newly created inquiry (`status: created`) and one for a completed inquiry (`status: COMPLETED`). Use them to test webhook handling:

```bash
curl -X POST http://localhost:3000/api/webhook \
  -H "Content-Type: application/json" \
  -H "Persona-Signature: <computed-hmac>" \
  -d @response.txt
```

---

## License

MIT License

Copyright (c) 2026 Muya Allan

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
