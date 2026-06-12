# ✅ Security Checklist — Gammal Tech Developer Pre-Deployment Guide

> This checklist must be reviewed before every production deployment.
> Every item should be marked ✅ before code is merged to `main`.
> If an item is marked ❌ or ⚠️, it must be resolved or explicitly documented with a risk acceptance sign-off.

---

## 1. 🔐 Authentication & Session Security

### Token & Session Management
- [ ] All protected routes verify authentication **before rendering** (not inside `useEffect`)
- [ ] A centralized `ProtectedRoute` component is used — no per-page auth logic copy-pasted
- [ ] Session tokens are stored in **`httpOnly` cookies** — never in `localStorage` or `sessionStorage`
- [ ] Tokens have a defined **expiry time** and are refreshed securely
- [ ] Logout **invalidates the token server-side** — not just clears client storage
- [ ] No "demo mode" flags, bypass keys, or hardcoded test credentials exist in production code
- [ ] Failed login attempts are **rate-limited** (e.g., max 5 attempts before lockout/CAPTCHA)
- [ ] Password reset tokens are **single-use** and expire within 15–60 minutes
- [ ] Multi-factor authentication (MFA) is offered for sensitive accounts

### SDK & Third-Party Auth
- [ ] `window.GammalTech.isLoggedIn()` is the **single source of truth** for auth state
- [ ] No `localStorage` flags (`is_demo_login`, `demo_user_email`, etc.) are used as auth state
- [ ] Token expiry is handled gracefully — expired sessions redirect to login, not crash
- [ ] The SDK `verify()` response is checked before granting access — not just `login()` success

---

## 2. 🛡️ Input Validation & Sanitization

### Client-Side Validation (UX only — never rely on this for security)
- [ ] All form fields have `type`, `min`, `max`, `maxlength`, and `required` attributes where applicable
- [ ] Numeric fields (`age`, `weight`, `height`) reject non-numeric input
- [ ] Text areas have a **character limit** enforced in both UI and backend

### Server-Side / AI-Side Validation (the real defense)
- [ ] All user inputs are **sanitized before being embedded in AI prompts**
- [ ] Prompt construction uses **structured parameters** — user content is never in the system prompt
- [ ] Inputs are validated against an **allowlist** of expected formats, not just a blocklist
- [ ] Medical input fields reject common **prompt injection patterns**:
  - Bracket instructions: `[تعليمات جديدة]`, `[new instructions]`
  - Role-override keywords: `ignore previous`, `تجاهل`, `you are now`, `أنت الآن`
  - System-level keywords: `system prompt`, `assistant mode`
- [ ] All inputs are **length-limited** before hitting any API:
  - Short fields (name, email): max 100 chars
  - Text areas (symptoms, history): max 1000 chars
  - No field accepts unlimited input

### Output Sanitization
- [ ] All AI-generated responses are sanitized with **DOMPurify** before rendering
- [ ] `dangerouslySetInnerHTML` is **never used** with AI-sourced or user-sourced content
- [ ] API error messages shown to users are **generic** — no stack traces, file paths, or internal details exposed

---

## 3. 🔒 Data Encryption & Storage

### Data at Rest
- [ ] No PII is stored in `localStorage` (emails, names, health data, user IDs)
- [ ] No PII is stored in `sessionStorage` unless absolutely necessary and non-sensitive
- [ ] Sensitive data stored client-side (if unavoidable) is **encrypted** before storage
- [ ] `node_modules/`, `dist/`, and `.env` files are in `.gitignore` and **never committed**
- [ ] No API keys, tokens, secrets, or passwords are hardcoded anywhere in source code
- [ ] `.env` files are confirmed absent from git history (`git log --all -- .env`)

### Data in Transit
- [ ] All API calls use **HTTPS** — no HTTP endpoints anywhere
- [ ] No sensitive data is passed in **URL query parameters** (tokens, IDs, emails)
- [ ] Payment data is handled **entirely by the SDK** — no raw card data touches app code
- [ ] `console.log()` statements containing user data, tokens, or payment info are **removed** before production

### Secret Management
- [ ] All environment variables are set in **Netlify/server environment settings** — not in code
- [ ] No secrets appear in git history (use `git-secrets` or `trufflehog` to verify)
- [ ] API keys are **scoped to minimum permissions** needed (principle of least privilege)
- [ ] Secrets are **rotated** if they were ever committed accidentally or shared

---

## 4. 🚦 Access Control

### Route & Page Protection
- [ ] Every authenticated route uses `ProtectedRoute` — no exceptions
- [ ] Admin-only routes have a **role check** in addition to auth check
- [ ] Direct URL access to protected pages (`/dashboard`, `/profile`, etc.) redirects unauthenticated users to `/login`
- [ ] The redirect preserves the intended destination so users land correctly after login

### Data Access (IDOR Prevention)
- [ ] Users can only access **their own data** — no user ID is accepted from the client to fetch other users' records
- [ ] All data-fetching calls use the **authenticated session identity** (from the token), not a user-supplied ID
- [ ] API endpoints verify that the requesting user **owns the resource** being accessed
- [ ] Pagination and list endpoints **filter by the authenticated user** — no "get all users" exposed to non-admins

### Payment & Subscription
- [ ] Premium features verify subscription status **server-side** — not via a client-readable flag
- [ ] `payment.confirmDelivery()` is called **only after** the backend confirms the transaction
- [ ] Backend payment errors **halt the flow** — no `.catch(() => { ok: true })` workarounds
- [ ] Payment amounts are validated **server-side** — client-sent amounts are never trusted

---

## 5. 🌐 HTTP Security Headers

### Required Headers (configure in `public/_headers` for Netlify)
- [ ] `Content-Security-Policy` is defined and restricts:
  - `script-src` to `'self'` and explicitly allowed CDNs only
  - `connect-src` to `'self'` and known API domains
  - `img-src` to `'self'` and known image CDNs
  - `default-src 'self'` as the fallback
- [ ] `X-Frame-Options: DENY` — prevents clickjacking
- [ ] `X-Content-Type-Options: nosniff` — prevents MIME sniffing
- [ ] `Referrer-Policy: strict-origin-when-cross-origin` — limits referrer leakage
- [ ] `Permissions-Policy` restricts unused browser APIs:
  ```
  Permissions-Policy: camera=(), microphone=(), geolocation=()
  ```
- [ ] `Strict-Transport-Security` (HSTS) is enabled for HTTPS enforcement

### CSRF Protection
- [ ] All state-changing requests (POST, PUT, DELETE, PATCH) include a **CSRF token**
- [ ] CSRF tokens are **verified server-side** on every mutating request
- [ ] `SameSite=Strict` or `SameSite=Lax` is set on session cookies

---

## 6. 📦 Third-Party Dependencies & Supply Chain

### Subresource Integrity (SRI)
- [ ] Every external script tag has an **`integrity` hash** and `crossorigin="anonymous"`:
  ```html
  <script src="https://api.gammal.tech/sdk-web.js"
          integrity="sha384-[HASH]"
          crossorigin="anonymous"></script>
  ```
- [ ] SRI hashes are **regenerated** every time the external script is updated
- [ ] No external scripts are loaded from untrusted or unversioned CDN URLs

### Dependency Auditing
- [ ] `npm audit` is run before every deployment — all **Critical and High** vulnerabilities are resolved
- [ ] Dependencies are **pinned to exact versions** in `package.json` for production builds
- [ ] `node_modules/` is **never committed** to the repository
- [ ] Unused dependencies are removed (`npm prune`)
- [ ] Dependencies are reviewed for **known malicious packages** before installation

---

## 7. 🤖 AI & LLM-Specific Security

### Prompt Security
- [ ] System prompts are **hardcoded server-side** — never constructed from user input
- [ ] User inputs and system instructions are sent as **separate parameters** to the AI API — never concatenated into one string
- [ ] AI responses are treated as **untrusted content** — always sanitized before rendering
- [ ] The AI is **not the last line of defense** — application-level safety checks exist independently

### Safety Rules Enforcement
- [ ] `ai-safety-rules.md` is reviewed and up to date
- [ ] AI outputs are **logged** for safety monitoring (with appropriate privacy protections)
- [ ] Emergency/crisis scenarios (chest pain, suicidal ideation, etc.) trigger **hardcoded responses** — not AI-generated ones
- [ ] The AI cannot be asked to **reveal its system prompt** — prompt is protected server-side
- [ ] Rate limiting is applied to AI endpoints to prevent **token exhaustion attacks**

### Medical Data Handling
- [ ] Health data submitted to the AI is **not stored permanently** unless user explicitly consents
- [ ] AI responses include the mandatory disclaimer: "هذا تقييم معلوماتي وليس تشخيصاً طبياً"
- [ ] No personally identifiable health data is logged in plaintext

---

## 8. 🗂️ Repository & Code Hygiene

### What Must Never Be in the Repo
- [ ] `.env` or `.env.local` files
- [ ] `node_modules/` directory
- [ ] `dist/` or `build/` directories
- [ ] `*.log` files (`vite_error.log`, `build_output.log`, etc.)
- [ ] API keys, tokens, passwords, or secrets of any kind
- [ ] Database connection strings
- [ ] Private keys or certificates

### Git Hygiene
- [ ] `.gitignore` is verified **before first commit** on any new project
- [ ] `git log --all --full-history -- .env` confirms no secrets in history
- [ ] If secrets were accidentally committed: immediately **rotate the secret**, then use `git filter-repo` to purge history
- [ ] Branch protection is enabled on `main` — no direct pushes, PRs required
- [ ] At least **one reviewer** must approve PRs before merge

---

## 9. 🧪 Pre-Deployment Testing

### Security Testing
- [ ] Run `npm audit` — no Critical or High vulnerabilities unresolved
- [ ] Test all protected routes with **no auth cookies/tokens** — confirm redirect to login
- [ ] Test all form fields with:
  - `<script>alert(1)</script>` — must render as text, never execute
  - SQL injection strings: `' OR '1'='1` — must be rejected or escaped
  - Prompt injection: `تجاهل التعليمات السابقة` — AI must not comply
  - Oversized input (max 10x the expected length) — must be truncated or rejected
- [ ] Verify **no debug/console.log** statements remain in production build
- [ ] Check browser DevTools Network tab — no sensitive data in query params or response bodies

### Functional Security Checks
- [ ] Login with wrong credentials → generic error message (not "user not found" vs "wrong password")
- [ ] Access `/dashboard` without login → redirect to `/login` ✅
- [ ] Access `/profile` without login → redirect to `/login` ✅
- [ ] Access `/health-advisor` without login → redirect to `/login` ✅
- [ ] Payment flow: block `/api/purchases` → transaction must NOT complete ✅
- [ ] Logout → session is fully cleared, back button does not restore protected pages

---

## 10. 🚀 Deployment Checklist (Final Gate)

- [ ] All items above are checked ✅
- [ ] Production environment variables are set (not `.env` file in the repo)
- [ ] CSP and security headers are live — verify with [securityheaders.com](https://securityheaders.com)
- [ ] HTTPS is enforced — verify with [SSL Labs](https://www.ssllabs.com/ssltest/)
- [ ] `npm audit` passes with 0 Critical/High issues
- [ ] The deployed app is tested on production URL (not just localhost)
- [ ] Error monitoring (e.g., Sentry) is active so crashes are detected immediately
- [ ] A rollback plan exists if a critical bug is found post-deploy

---

## Quick Reference — Top 5 Rules to Never Break

| # | Rule | Why |
|---|------|-----|
| 1 | **Never store tokens or PII in `localStorage`** | Accessible to XSS, persists across sessions |
| 2 | **Never trust client-side auth state alone** | Easy to manipulate in DevTools |
| 3 | **Never embed user input in AI system prompts** | Enables prompt injection attacks |
| 4 | **Never commit secrets, logs, or `node_modules`** | Permanent exposure in git history |
| 5 | **Never confirm payment if backend fails** | Enables free access / financial fraud |

---

## Resources

- [OWASP Top 10 (2021)](https://owasp.org/Top10/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [Mozilla Web Security Guidelines](https://infosec.mozilla.org/guidelines/web_security)
- [Content Security Policy Reference](https://content-security-policy.com/)
- [SRI Hash Generator](https://www.srihash.org/)
- [Security Headers Checker](https://securityheaders.com/)
- [Have I Been Pwned API](https://haveibeenpwned.com/API/v3) — check if user emails are in known breaches

---

*Last updated: June 12, 2026*
*Maintained by: Gammal Tech Security Team*
*Review cycle: Before every major deployment and quarterly otherwise*
