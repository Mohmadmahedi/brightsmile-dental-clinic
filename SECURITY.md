# 🛡️ Security Audit & Backend Architecture Report
**Project:** BrightSmile Dental Clinic  
**Audit Date:** October 2026  
**Audited Components:** Frontend (`index.html`), Backend Architecture (`server.js`), API Keys & Secrets Management (`.env.example`, `.gitignore`).

---

## 1. 🔑 API Key & Secrets Audit

| Check | Status | Details |
|---|---|---|
| **Hardcoded API Keys in Frontend** | ✅ **PASSED** | Full regex and string audit found **zero** exposed API keys, secret tokens, private keys, or passwords in `index.html`. |
| **Maps API Exposure** | ✅ **PASSED** | Uses secure Google Maps web protocol links and an SVG canvas mockup. Zero leaked Google Cloud / Maps API keys. |
| **Database Credentials** | ✅ **PASSED** | No database connection strings or credentials in client code. |
| **Git Leak Protection** | ✅ **PROTECTED** | `.gitignore` explicitly prevents `.env`, `.pem`, and secret files from being committed to source control. |

---

## 2. 🌐 Frontend Security Controls (`index.html`)

### A. Cross-Site Scripting (XSS) Prevention
- **DOM Insertion:** All dynamic user values (e.g. `confirmPatientName`, `confirmPatientPhone`, `summaryService`) are bound using `node.textContent` rather than `innerHTML`. This guarantees that user input is treated strictly as plain text, neutralising DOM-based XSS attacks.
- **Client Sanitization:** Inputs are trimmed and length-capped (`maxlength="80"` for names, `maxlength="25"` for phones, `maxlength="100"` for emails, `maxlength="1000"` for messages).

### B. Content Security Policy (CSP) & Headers
- Embedded in `<head>` via `<meta http-equiv="Content-Security-Policy">`:
  ```http
  default-src 'self';
  script-src 'self' 'unsafe-inline';
  style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
  font-src 'self' https://fonts.gstatic.com;
  img-src 'self' data: https://images.unsplash.com;
  connect-src 'self' http://localhost:* https:;
  frame-ancestors 'none';
  ```
- **Anti-Clickjacking:** `frame-ancestors 'none'` blocks attackers from embedding the site within malicious `<iframe>` overlays.
- **MIME Sniffing Protection:** `<meta http-equiv="X-Content-Type-Options" content="nosniff">` prevents browsers from misinterpreting non-executable MIME types.
- **Referrer Privacy:** `<meta name="referrer" content="strict-origin-when-cross-origin">` prevents leaking internal query parameters to external sites.

### C. Anti-Spam Bot Honeypot
- A zero-pixel hidden field (`clinic_hp_field`) with `tabindex="-1"` and `autocomplete="off"` is included in the form.
- Automated bots and scrapers fill this field blindly; submissions with this field populated are dropped silently without alerting spammers.

### D. Reverse Tabnabbing Protection
- All external links (such as WhatsApp, Maps, and social links) include:
  ```html
  target="_blank" rel="noopener noreferrer"
  ```
  This prevents target pages from accessing the `window.opener` object.

---

## 3. ⚙️ Backend Security Architecture (`server.js`)

For production deployment, the backend server implements defense-in-depth protections:

1. **Helmet.js Security Suite:** Automatically sets HTTP Strict Transport Security (HSTS), X-XSS-Protection, X-Frame-Options (`DENY`), and X-Content-Type-Options.
2. **Rate Limiting:**
   - Global API limit: 100 requests per 15 minutes per IP.
   - `/api/appointments` limit: **5 requests per 15 minutes per IP** to eliminate appointment spam, SMS bombing, and automated flooding.
3. **Payload Size Restrictions:** `express.json({ limit: '20kb' })` prevents memory exhaustion and buffer overflow attacks.
4. **CORS Restrictions:** Only requests originating from approved domains specified in `ALLOWED_ORIGINS` are accepted.
5. **Server-Side Input Sanitization:**
   - Strips dangerous characters (`<`, `>`).
   - Validates regex patterns for phone numbers and email formats.
   - Enforces strict character limits before data persistence.
6. **Information Disclosure Prevention:**
   - No database errors or stack traces are leaked to clients.
   - Generic sanitized error responses are returned (`500 Internal Server Error`).

---

## 4. 🏥 Healthcare & Patient Privacy (HIPAA / GDPR Best Practices)

If deploying this system with real patient data:
1. **Encryption in Transit:** Enforce HTTPS/TLS 1.3 across all domains.
2. **Encryption at Rest:** Ensure database storage encrypts patient phone numbers and emails using AES-256.
3. **Audit Logging:** Keep access logs for staff reviewing appointment requests.
4. **Ephemeral Patient Messages:** Do not store sensitive medical history in plain text email notifications. Use portal links or encrypted notification services.
