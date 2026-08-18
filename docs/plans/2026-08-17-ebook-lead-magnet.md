# Ebook Lead Magnet (email-gated download) Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** A visitor enters their email on prismera.com.au, receives an email with a one-time download link to an ebook PDF, and the lead (with UTM attribution) lands in ERPNext.

**Architecture:** Extend the existing `prismera-proxy` (localhost:8955, behind the Cloudflare quick tunnel) with two new routes: `POST /ebook/submit` (validate email, create ERPNext Lead, store a one-time token in Postgres, email the link) and `GET /ebook/download/<token>` (validate token, mark used, stream the PDF). A new static page `src/ebook/index.html` hosts the offer form. No new infrastructure beyond a local Postgres database for tokens.

```
prismera.com.au/ebook/ (GH Pages, form JS)
  | POST /ebook/submit (email + UTM params + honeypot)
  v
<random>.trycloudflare.com (existing quick tunnel)
  v
localhost:8955 proxy.py (existing prismera-proxy systemd unit)
  |-- validate + honeypot + rate limit
  |-- create ERPNext Lead via REST API (utm_campaign link field)
  |-- insert token into Postgres ebook_downloads (utm as JSONB)
  |-- SMTP (mailcow) -> visitor email with https://<tunnel>/ebook/download/<token>
  |-- GET /ebook/download/<token> -> validate, mark used, stream PDF
```

**Tech Stack:** Python http.server (existing proxy, no new framework), PostgreSQL (Tom's standard; JSONB for UTM), SMTP via mailcow (mail.prismera.com.au), ERPNext REST API (`/api/resource/Lead`), GitHub Pages static site.

---

## Current context (verified)

- Website: `/home/tom/projects/prismera-website`, deploys from `src/` on `main` to https://prismera.com.au (GH Actions `static.yml`).
- Proxy: `/home/tom/projects/prismera-proxy/proxy.py`, systemd user unit `prismera-proxy`, listens on localhost:8955.
- Tunnel: Cloudflare quick tunnel (`prismera-quicktunnel.service`), URL cached in `~/.prismera-tunnel-url.txt`, watchdog `/home/tom/.local/bin/prismera-tunnel-watch.sh` (every 3 min) syncs the URL into `src/contact/index.html` (line ~268 fetch) and pushes.
- Contact form flow (health-check skill): proxy POST `/submit` -> Gmail SMTP notification -> Hermes webhook localhost:8644. Health-check steps in `prismera-email-health-check` skill are the verification template.
- ERPNext (erp.prismera.com.au): Lead doctype has `utm_campaign` (Link to UTM Campaign), `utm_source`, `utm_medium`, `utm_content` fields. A UTM Campaign record must exist before a Lead can link to it.
- Site style rules: no em-dashes anywhere in copy; header/footer are inlined into every page (editing templates alone changes nothing).

## Key decisions (defaults chosen, each reversible)

1. **Delivery mechanism: one-time link, not attachment.** Proxy stores a token and emails a link (`/ebook/download/<token>`). Prevents link sharing, gives a download event, avoids attachment stripping. Single use, expires after 7 days.
2. **Token store: PostgreSQL** (Tom's stated standard). Table `ebook_downloads` with `utm` as JSONB. Requires a local Postgres on the proxy host (check `pg_isready`; install if absent).
3. **Sender: mailcow SMTP** (mail.prismera.com.au:587, STARTTLS, From hello@prismera.com.au). Proper branding and SPF/DKIM on the prismera.com.au domain. Gmail SMTP is the fallback if mailcow is unreachable, with the caveat that Gmail rewrites From.
4. **Lead capture: ERPNext Lead via REST**, created by the proxy, with UTM fields set. Uses a dedicated API user (Tom creates it in the ERPNext UI and enters the key himself, per his standing rule on secrets).
5. **Offer placement: dedicated page `/ebook/` + homepage CTA card.** Nav link optional (13 inlined headers to touch).

---

## Task 1: Placeholder ebook PDF

**Objective:** Have a real file on disk the proxy can serve, so every later task is testable.

**Files:**
- Add: `/home/tom/projects/prismera-proxy/ebooks/prismera-ebook.pdf`

**Steps:**

1. Create the dir: `mkdir -p /home/tom/projects/prismera-proxy/ebooks`
2. Put a placeholder PDF at `ebooks/prismera-ebook.pdf` (a generated 1-page PDF is fine: `python3 -c "..."` with reportlab if installed, else any small PDF). The real ebook is a Tom-supplied asset; the config points at this path so swapping the file later is a zero-code change.
3. Note in the plan's open questions that the final PDF content is pending Tom.

**Verification:** `ls -la /home/tom/projects/prismera-proxy/ebooks/` shows the file, `file ebooks/prismera-ebook.pdf` reports PDF.

---

## Task 2: Proxy config file

**Objective:** All new settings live in one JSON config, secrets excluded from git.

**Files:**
- Add: `/home/tom/projects/prismera-proxy/ebook_config.json` (chmod 600)
- Modify: `/home/tom/projects/prismera-proxy/.gitignore` (add `ebook_config.json` if not already ignored)

**Content (placeholders Tom fills for secrets):**

```json
{
  "ebook_path": "/home/tom/projects/prismera-proxy/ebooks/prismera-ebook.pdf",
  "ebook_name": "Prismera Ebook",
  "link_expiry_days": 7,
  "smtp": {
    "host": "mail.prismera.com.au",
    "port": 587,
    "user": "hello@prismera.com.au",
    "password": "PENDING-TOM",
    "from": "hello@prismera.com.au"
  },
  "erpnext": {
    "base_url": "https://erp.prismera.com.au",
    "api_key": "PENDING-TOM",
    "api_secret": "PENDING-TOM",
    "utm_campaign_name": "Ebook Download"
  },
  "database": {
    "dsn": "postgresql://ebook:CHANGE_ME@localhost:5432/prismera_ebook"
  }
}
```

**Step: secrets handling** (Tom's rule: he enters secrets himself in the target app UI, never pastes them in chat). Tom fills `smtp.password` (mailcow mailbox password) and the ERPNext `api_key`/`api_secret` after Task 11 creates the API user. The proxy must fail loudly (log + 500) if any `PENDING-TOM` value remains at startup.

**Verification:** `python3 -c "import json; json.load(open('/home/tom/projects/prismera-proxy/ebook_config.json'))"` parses clean; file mode is 600.

---

## Task 3: PostgreSQL database

**Objective:** Token store table with Tom's conventions (real Postgres, JSONB for the UTM payload).

**Files:**
- Add: `/home/tom/projects/prismera-proxy/init.sql`

**Steps:**

1. Check for an existing local Postgres: `pg_isready` (exit 0 means present). If absent, `sudo apt install postgresql` (Ubuntu packages postgres 16+; the proxy host is the Hermes workstation, so this is a one-time install).
2. Create the role and database (as the postgres superuser):
   `sudo -u postgres psql -c "CREATE ROLE ebook LOGIN PASSWORD 'CHANGE_ME';" -c "CREATE DATABASE prismera_ebook OWNER ebook;"`
3. Apply `init.sql`:
   `sudo -u postgres psql -d prismera_ebook -f /home/tom/projects/prismera-proxy/init.sql`

```sql
CREATE TABLE IF NOT EXISTS ebook_downloads (
    id          bigserial PRIMARY KEY,
    email       varchar(255) NOT NULL,
    token       varchar(64)  NOT NULL UNIQUE,
    utm         jsonb        NOT NULL DEFAULT '{}'::jsonb,
    created_at  timestamptz  NOT NULL DEFAULT now(),
    expires_at  timestamptz  NOT NULL,
    used_at     timestamptz  NULL,
    downloaded  boolean      NOT NULL DEFAULT false
);
CREATE INDEX IF NOT EXISTS idx_ebook_downloads_email ON ebook_downloads (email);
```

**Verification:** `sudo -u postgres psql -d prismera_ebook -c "\d ebook_downloads"` shows the table with the jsonb column.

---

## Task 4: Proxy `POST /ebook/submit`

**Objective:** Accept the form, validate, create the ERPNext Lead, store the token, send the email.

**Files:**
- Modify: `/home/tom/projects/prismera-proxy/proxy.py`
- Add: `/home/tom/projects/prismera-proxy/tests/test_ebook.py`

**Step 1: Write failing tests** (Tom's rule: tests use real PostgreSQL, never SQLite; a pytest fixture creates a throwaway DB via the same `init.sql`):

```python
def test_submit_valid_email_creates_token_and_lead(fake_smtp, fake_erpnext, test_db):
    code, body = post_ebook_submit({"email": "test@example.com", "utm_source": "google"})
    assert code == 200
    assert fake_smtp.last_to == "test@example.com"
    assert fake_erpnext.last_lead["email_id"] == "test@example.com"
    assert fake_erpnext.last_lead["utm_source"] == "google"
    assert "download" in fake_smtp.last_body

def test_submit_rejects_bad_email(fake_smtp, test_db):
    assert post_ebook_submit({"email": "nope"})[0] == 400

def test_submit_honeypot_rejected(fake_smtp, test_db):
    assert post_ebook_submit({"email": "t@t.com", "website": "spam"})[0] == 400

def test_submit_rate_limit_per_ip(test_db):
    for _ in range(5): post_ebook_submit({"email": "t@t.com"})
    assert post_ebook_submit({"email": "t@t.com"})[0] == 429
```

**Step 2: Run tests, expect FAIL** (`pytest tests/test_ebook.py -v`, endpoints missing).

**Step 3: Implement the handler.** Add to `proxy.py` (it already parses POST JSON for `/submit`; reuse that plumbing):

- `POST /ebook/submit`: parse `{email, utm_source, utm_medium, utm_campaign, utm_content, website}` (honeypot field).
- Validate: `email` matches a simple regex; `website` must be empty (honeypot); per-IP rate limit (in-memory dict, 5/hour, 429).
- Dedupe: check ERPNext for an existing Lead with that `email_id` (`GET /api/resource/Lead?filters=[["email_id","=","..."]]`); if found, still send the email but skip the new Lead.
- Create the Lead: `POST /api/resource/Lead` with `{lead_name, email_id, source, utm_source, utm_medium, utm_campaign: <utm_campaign_name>, utm_content}`. The `utm_campaign` value must be the UTM Campaign record name (created in Task 11), because the field is a Link. Use the API key auth header `Authorization: token <api_key>:<api_secret>`.
- Generate token: `secrets.token_urlsafe(32)`.
- Insert into `ebook_downloads` with `expires_at = now() + interval '7 days'`, utm as JSONB.
- Send email via SMTP (mailcow, STARTTLS, `smtplib`), body contains `https://<tunnel_url>/ebook/download/<token>` (tunnel URL read from `~/.prismera-tunnel-url.txt` at send time; if the file is missing, fall back to a configurable base URL and log a warning).
- Respond `{"status": "ok"}` with CORS headers (same pattern as `/submit`).

**Step 4: Run tests, expect PASS.**

**Step 5: Commit** in the proxy repo: `git add proxy.py tests/ && git commit -m "feat: ebook submit endpoint"`.

---

## Task 5: Proxy `GET /ebook/download/<token>`

**Objective:** Serve the PDF once, only for a valid, unexpired, unused token.

**Files:**
- Modify: `/home/tom/projects/prismera-proxy/proxy.py`
- Add: `/home/tom/projects/prismera-proxy/tests/test_ebook_download.py`

**Step 1: Failing tests:**

```python
def test_download_valid_token_streams_pdf(fake_ebook_file, test_db):
    token = insert_token(test_db, "t@t.com")
    code, headers, body = get_ebook_download(token)
    assert code == 200
    assert headers["Content-Type"] == "application/pdf"
    assert body == fake_ebook_file_bytes

def test_download_second_use_rejected(fake_ebook_file, test_db):
    token = insert_token(test_db, "t@t.com")
    get_ebook_download(token)
    assert get_ebook_download(token)[0] == 410

def test_download_expired_rejected(fake_ebook_file, test_db):
    token = insert_token(test_db, "t@t.com", expired=True)
    assert get_ebook_download(token)[0] == 410

def test_download_unknown_token_404(test_db):
    assert get_ebook_download("bogus")[0] == 404
```

**Step 2: Run, expect FAIL.**

**Step 3: Implement.** Route regex `/ebook/download/([A-Za-z0-9_-]+)`: lookup by token; 404 if missing; 410 if `downloaded` or past `expires_at`; otherwise mark `used_at`/`downloaded` in a transaction, then stream the file bytes with `Content-Type: application/pdf` and `Content-Disposition: attachment; filename="prismera-ebook.pdf"`. Read the configured `ebook_path`; 500 if the file is missing on disk.

**Step 4: Run, expect PASS. Step 5: Commit** `feat: ebook download endpoint`.

---

## Task 6: Restart and local verification

**Objective:** The live proxy serves both routes.

**Steps:**

1. `systemctl --user restart prismera-proxy`
2. `curl -s -X POST http://localhost:8955/ebook/submit -H 'Content-Type: application/json' -d '{"email":"test@example.com"}'` -> `{"status":"ok"}` (check journal for SMTP errors if not).
3. `curl -s -o /dev/null -w "%{http_code}" http://localhost:8955/ebook/download/<token-from-db>` -> 200; second call -> 410.
4. `journalctl --user -u prismera-proxy --since "2 minutes ago" --no-pager` shows no tracebacks.

---

## Task 7: Website page `src/ebook/index.html`

**Objective:** The offer page with the email form.

**Files:**
- Add: `src/ebook/index.html`
- Add: `src/assets/ebook.js` (form handler, referenced by the page)

**Steps:**

1. Copy the skeleton from `src/about-us/index.html` (inlined header + footer, hero block). Reuse the `.blog-article` style block pattern from `src/blog/m365-defaults/index.html`.
2. Content: H1 with the ebook value proposition, 2-3 bullet benefits, the form (single email input + submit button + microcopy "We only use your email to send the ebook. See our [privacy policy](/privacy-policy/)."), success message ("Check your inbox") and error message states.
3. Form JS in `src/assets/ebook.js`:

```javascript
const EBOOK_PROXY_URL = 'https://<tunnel-url>/ebook/submit';
document.getElementById('ebook-form').addEventListener('submit', async (e) => {
  e.preventDefault();
  const q = new URLSearchParams(window.location.search);
  const payload = {
    email: document.getElementById('ebook-email').value.trim(),
    website: document.getElementById('ebook-website').value, // honeypot
    utm_source: q.get('utm_source'),
    utm_medium: q.get('utm_medium'),
    utm_campaign: q.get('utm_campaign'),
    utm_content: q.get('utm_content')
  };
  const res = await fetch(EBOOK_PROXY_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload)
  });
  // res.ok -> show success state; else show error state
});
```

The `<tunnel-url>` placeholder is replaced by the watchdog (Task 9), same as the contact page pattern.
4. Add the GA4 `dataLayer.push({event: 'ebook_submit'})` on success (the head already has GA4; optional but cheap).
5. No em-dashes in copy: `grep -n "—" src/ebook/index.html` must be empty.

**Verification:** `python3 -m http.server` from `src/` locally and confirm the page renders, or after deploy (Task 13) curl the live URL.

---

## Task 8: Homepage CTA + optional nav link

**Objective:** Traffic paths into the offer.

**Files:**
- Modify: `src/index.html` (CTA card after the hero or in the services section)
- Optional: nav link in all 13 inlined headers + `src/_header.html` template

**Steps:**

1. Add a CTA card to `src/index.html` linking to `/ebook/` ("Get the ebook" style, matching existing card styling).
2. Optional nav: insert `<a href="/ebook/" class="md-nav__link" data-page="ebook">...</a>` into the header block on all pages; verify coverage with `grep -rc 'data-page="ebook"' src/ --include="*.html" | grep -v ":0" | wc -l` -> 14 (13 headers + the new page's own header).

**Verification:** grep counts above; page link renders.

---

## Task 9: Tunnel watchdog covers the ebook page

**Objective:** The quick-tunnel URL keeps working for the ebook form without manual edits.

**Files:**
- Modify: `/home/tom/.local/bin/prismera-tunnel-watch.sh`

**Steps:**

1. The script currently rewrites the tunnel URL in `src/contact/index.html`. Change the target discovery to `grep -rl 'trycloudflare.com/ebook/submit' src/` plus the contact page, and sed the URL into all matching pages (contact and ebook).
2. Keep the commit + push step as-is (same `GITHUB_TOKEN` from `~/.hermes/.env`).

**Verification:** `bash /home/tom/.local/bin/prismera-tunnel-watch.sh` updates both files; `grep -c trycloudflare.com/ebook/submit src/ebook/index.html` shows the new URL; the commit contains both pages.

---

## Task 10: Email template

**Objective:** A deliverable, on-brand email with the link.

**Files:**
- Modify: `/home/tom/projects/prismera-proxy/proxy.py` (template strings)

**Steps:**

1. Subject: "Your Prismera ebook download".
2. Plain-text body: greeting, one line on the ebook, the download link, a line that the link works once and expires in 7 days, and "reply to this email or email hello@prismera.com.au to unsubscribe".
3. Optional simple HTML alternative (same content). No attachments.
4. Send with `smtplib.SMTP` + STARTTLS to mailcow; `From: hello@prismera.com.au`, `To: <visitor>`.

**Verification:** Task 12 end-to-end covers it; also verify SPF/DKIM alignment in the received header (mailcow signs the domain).

---

## Task 11: ERPNext prep (Tom does the UI parts)

**Objective:** A Lead can be created with UTM attribution.

**Steps:**

1. Tom creates the UTM Campaign record "Ebook Download" in ERPNext (Marketing > UTM Campaign). This is required because `Lead.utm_campaign` is a Link field; the proxy sends this name.
2. Tom creates an API user (e.g. `apiproxy@prismera.com.au`, System Manager or a scoped role) and generates API keys (User settings > API Access > Generate Keys).
3. Tom pastes the key/secret into `/home/tom/projects/prismera-proxy/ebook_config.json` (Task 2 placeholders).
4. Confirm the Lead fields the proxy sets (`lead_name`, `email_id`, `source`, `utm_*`) exist and note the accepted `source` values (check via the ERPNext UI or `frappe.db.sql("SELECT DISTINCT source FROM tabLead")`).

**Verification:** a test Lead created by the proxy (Task 12) shows the UTM Campaign link in the ERPNext UI.

---

## Task 12: End-to-end verification

**Objective:** Prove the whole funnel works, following the health-check skill.

**Steps:**

1. Proxy endpoint: `curl -s -X POST http://localhost:8955/ebook/submit -d '{"email":"<tom-address>"}'` -> `{"status":"ok"}`.
2. Public tunnel: same POST to `$(cat ~/.prismera-tunnel-url.txt)/ebook/submit` -> 200.
3. Inbox: email arrives from hello@prismera.com.au with the link (allow a minute).
4. Download: click/curl the link -> 200 PDF; second click -> 410.
5. ERPNext: the Lead exists with `email_id`, `utm_campaign` = "Ebook Download", and the utm fields set.
6. Anti-abuse: bad email -> 400, honeypot filled -> 400, 6th rapid submit -> 429.
7. Deliverability spot check: header shows SPF/DKIM pass for prismera.com.au (mailcow signing).

---

## Task 13: Deploy and live check

**Objective:** The offer is live on prismera.com.au.

**Steps:**

1. In `/home/tom/projects/prismera-website`: `git add -A && git commit -m "feat: ebook lead magnet page"`.
2. Push with the token (no gh CLI on this host):
   `bash -c 'source ~/.hermes/.env && git push https://tom-hooper:$GITHUB_TOKEN@github.com/tom-hooper/prismera.git main'`
3. Poll the Actions API until `conclusion: success` (~2-4 min), then `curl -s https://prismera.com.au/ebook/ | grep -o 'ebook-email'` (or another marker). 404 right after push is normal while the workflow builds.
4. Repeat the Task 12 flow against the live site URL.

---

## Files likely to change (summary)

- `/home/tom/projects/prismera-proxy/proxy.py` (ebook submit + download routes, email template)
- `/home/tom/projects/prismera-proxy/ebook_config.json` (new, secrets, chmod 600, gitignored)
- `/home/tom/projects/prismera-proxy/init.sql` (new)
- `/home/tom/projects/prismera-proxy/tests/test_ebook.py`, `tests/test_ebook_download.py` (new)
- `/home/tom/projects/prismera-proxy/ebooks/prismera-ebook.pdf` (placeholder, Tom supplies real PDF)
- `/home/tom/projects/prismera-website/src/ebook/index.html` (new)
- `/home/tom/projects/prismera-website/src/assets/ebook.js` (new)
- `/home/tom/projects/prismera-website/src/index.html` (CTA card)
- 13 header files if the nav link is added (optional)
- `/home/tom/.local/bin/prismera-tunnel-watch.sh` (sync URL into ebook page too)
- ERPNext: UTM Campaign "Ebook Download" (Tom, UI), API user (Tom, UI)

## Risks and open questions

- **Ebook content:** the PDF is a Tom-supplied asset. Placeholder keeps the pipeline testable; content, title, and file name can change without code changes.
- **mailcow reachability from the proxy host:** if mail.prismera.com.au is unreachable (NAS on LAN, VPN), fall back to Gmail SMTP and accept From rewriting, or add the alias to Gmail. Flag before going live.
- **Postgres on the proxy host:** `pg_isready` first; install via apt if absent. No Docker needed on this box; if Docker is preferred later, the DSN in config is the only change.
- **ERPNext API user roles:** System Manager is simplest; a scoped role (Lead create + read) is tighter. Tom decides in the UI.
- **UTM campaign mapping:** the form's `utm_campaign` string is passed to ERPNext as the UTM Campaign record name. A mismatch (typo, or a campaign that doesn't exist) fails Lead creation; the proxy should log and degrade to creating the Lead without the link field rather than failing the download email.
- **Quick tunnel churn:** the tunnel URL changes on every restart; the watchdog covers the ebook page after Task 9, but there is a window until it runs (up to 3 min). Same exposure the contact form already has.
- **Privacy/compliance:** the form shows a consent line and links the privacy policy; the email offers an unsubscribe path. Good enough for this volume; a full double opt-in is a later option.
- **Rate limiting is in-memory:** resets on proxy restart. Fine at this traffic level.
