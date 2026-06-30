# Prismera Material Design Rebuild — Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** Rebuild the prismera.com.au website from scratch with Material Design principles, a teal/green color palette (no blue), professional real copy (no lorem ipsum), and a working contact form.

**Architecture:** Clean static HTML/CSS site with shared stylesheet and vanilla JS for the contact form. Single shared header/footer across all 6 pages. Deployed via existing GitHub Actions to GitHub Pages from `./src`.

**Tech Stack:** HTML5, CSS3 (Material Design via custom properties), vanilla JS, Google Material Icons (CDN)

**Current color scheme:** Blue/purple (#5733FF) → **New:** Teal/emerald green (#00897B primary)

---

## Pre-Flight: Current Issues Catalog

| Page | Issue |
|------|-------|
| **Home** | Lorem ipsum footer tagline, fake phone (+614****0000), social links→#, blog nav→#, contact nav→#, placeholder testimonial text |
| **About Us** | Title shows "About Us - Prismera" but content is placeholder ("This page needs updating"), lorem ipsum body |
| **Services** | Just says "Placeholder for services" |
| **Reviews** | "We're collecting testimonials" — no content |
| **Blog** | "Coming Soon" — no content |
| **Contact** | Title bug says "About Us - Prismera", phone is 0400 000 000 (fake), form posts to `/api/contact` but proxy expects `/submit` |
| **All pages** | Footer lorem ipsum, dead social/footer links, blue color scheme, bloated WP export (119KB home page), useless jQuery + emoji scripts |

---

## Tasks

### Phase 1: Scaffold & Stylesheet

#### Task 1: Clean the source directory

**Objective:** Remove the bloated WordPress export, preserving only needed assets

**Files:**
- Delete: `src/wp-content/`, `src/wp-includes/`
- Delete: `src/styles.css`, `src/index.html.bak`
- Preserve: `src/wp-content/uploads/2025/03/wN9cEWVb.png` (Prismera logo — move to `src/assets/logo.png`)

**Step 1:** Move logo to new location
```bash
mkdir -p src/assets
cp src/wp-content/uploads/2025/03/wN9cEWVb.png src/assets/logo.png
```

**Step 2:** Remove WP bloat
```bash
rm -rf src/wp-content src/wp-includes src/styles.css src/index.html.bak
```

**Step 3:** Commit
```bash
git add src/assets/logo.png && git rm -r src/wp-content src/wp-includes src/styles.css src/index.html.bak
git commit -m "chore: remove WordPress export bloat, preserve logo"
```

---

#### Task 2: Create shared stylesheet with Material Design tokens

**Objective:** Build a single CSS file with Material Design 3 color system, typography, elevation, and component styles — zero blue.

**Files:**
- Create: `src/css/material.css`

**Color palette:**
```css
:root {
  /* Primary — Teal (no blue) */
  --md-primary: #00897B;
  --md-primary-dark: #00695C;
  --md-primary-light: #4DB6AC;
  --md-on-primary: #FFFFFF;

  /* Secondary — Warm Amber */
  --md-secondary: #FF8F00;
  --md-secondary-dark: #E65100;
  --md-on-secondary: #FFFFFF;

  /* Surface & Background */
  --md-surface: #FFFFFF;
  --md-surface-variant: #F5F5F5;
  --md-background: #FAFAFA;
  --md-on-surface: #212121;
  --md-on-surface-variant: #616161;

  /* Status */
  --md-error: #D32F2F;
  --md-success: #2E7D32;

  /* Elevation */
  --md-elevation-1: 0 1px 3px rgba(0,0,0,0.12), 0 1px 2px rgba(0,0,0,0.08);
  --md-elevation-2: 0 3px 6px rgba(0,0,0,0.15), 0 2px 4px rgba(0,0,0,0.10);
  --md-elevation-3: 0 10px 20px rgba(0,0,0,0.15), 0 3px 6px rgba(0,0,0,0.10);
  --md-elevation-4: 0 14px 28px rgba(0,0,0,0.20), 0 5px 10px rgba(0,0,0,0.12);

  /* Typography — Material Design type scale */
  --md-font: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  --md-typescale-display: 3.562rem;
  --md-typescale-headline: 2.5rem;
  --md-typescale-title: 1.5rem;
  --md-typescale-body: 1rem;
  --md-typescale-label: 0.875rem;
  --md-typescale-small: 0.75rem;

  /* Shape */
  --md-shape-small: 8px;
  --md-shape-medium: 12px;
  --md-shape-large: 16px;
  --md-shape-full: 9999px;

  /* Layout */
  --md-max-width: 1200px;
  --md-header-height: 72px;
}
```

**Step 1:** Write the complete CSS file with:
- CSS reset
- Typography styles (h1–h6, p, body)
- Layout utilities (container, grid, flex helpers)
- Button component (.md-button, .md-button--filled, .md-button--outlined, .md-button--text)
- Card component (.md-card, .md-card--elevated)
- Header/navbar component
- Footer component
- Form inputs (text, email, tel, textarea)
- Hero section styles
- Pricing table styles
- Testimonial card styles
- Responsive breakpoints
- Ripple effect (CSS-only)
- Utility classes (text-center, spacing, etc.)

**Step 2:** Commit
```bash
git add src/css/material.css
git commit -m "feat: add Material Design 3 stylesheet with teal palette"
```

---

### Phase 2: Shared Components

#### Task 3: Create shared header component snippet

**Objective:** Build a reusable header with logo, navigation, and CTA button

**Design:**
- Fixed top bar, 72px height, white surface with elevation-1 shadow
- Logo (40×40px) + "Prismera" text on left
- Nav links: Home, About Us, Services, Reviews, Blog, Contact
- "Get Started" filled button (teal) on right
- Mobile: hamburger menu with slide-out drawer
- Active page link gets primary color + bottom border indicator

**Nav links (all real URLs):**
- Home → `/`
- About Us → `/about-us/`
- Services → `/services/`
- Reviews → `/reviews/`
- Blog → `/blog/`
- Contact → `/contact/`

**Step 1:** Create `src/_header.html` snippet
**Step 2:** Create mobile menu JS in `src/js/nav.js`
**Step 3:** Commit

---

#### Task 4: Create shared footer component snippet

**Objective:** Replace the lorem ipsum footer with real content and links

**Footer layout (4 columns):**
1. **Prismera** — Logo, real tagline: "Enterprise-grade IT services and digital transformation for Australian businesses."
2. **Services** — Managed IT, Cybersecurity, Cloud Solutions, Digital Strategy, Project Management
3. **Company** — About Us, Blog, Contact Us, Privacy Policy
4. **Contact** — hello@prismera.com.au, Melbourne, Australia

Bottom bar: "© 2025 Prismera Pty Ltd. All rights reserved."

**Step 1:** Create `src/_footer.html` snippet
**Step 2:** Commit

---

### Phase 3: Page Builds (each page = one task)

#### Task 5: Build Home page (`src/index.html`)

**Objective:** Professional landing page with hero, value props, pricing, and CTA

**Sections:**
1. **Hero** — Headline: "Enterprise IT, Simplified." Subhead: "Australia's managed technology partner — security, cloud, and strategy under one roof." Two buttons: "Get Started" (filled) and "Our Services" (outlined)
2. **Value Props** (3-column card grid) — icons + titles + descriptions for: Managed Services, Cybersecurity, Digital Strategy
3. **Why Prismera** (2-column) — Left: "Australian-based team, enterprise-grade security, proactive monitoring 24/7." Right: feature list with checkmarks
4. **Pricing** (3 cards) — **Starter** $99/mo (5+ users, Email + Basic Security), **Business** $249/mo (10+ users, Full Security Stack, Unlimited Support, Backup, vCIO), **Enterprise** $599/mo (Unlimited, Dedicated Staff, Custom Solutions)
5. **CTA Banner** — "Ready to secure your business?" with Get Started button
6. **Testimonials preview** (2 selected quotes → link to Reviews page)

**Copy rules:** No lorem ipsum. No "Unlock Tomorrow's Tech Today" cringe. Professional, direct, Australian tone.

**Step 1:** Write `src/index.html` with all sections
**Step 2:** Commit

---

#### Task 6: Build About Us page (`src/about-us/index.html`)

**Objective:** Real about page with company story, mission, and team placeholder

**Sections:**
1. **Hero banner** — "About Prismera" title with subtitle
2. **Our Story** — 2-3 paragraphs about Prismera being an Australian IT services company focused on making enterprise-grade technology accessible to growing businesses
3. **Mission & Values** — 3 cards: "Security First", "Australian Owned & Operated", "No Lock-In Contracts"
4. **CTA** — "Work with us" link to contact

**Step 1:** Write `src/about-us/index.html`
**Step 2:** Commit

---

#### Task 7: Build Services page (`src/services/index.html`)

**Objective:** Detailed services page replacing the "Placeholder" text

**Sections:**
1. **Hero** — "Our Services" 
2. **Service cards** (detailed, one per row):
   - **Managed IT Services** — Proactive monitoring, helpdesk support, hardware procurement, vendor management
   - **Cybersecurity** — Threat detection, endpoint protection, network security, compliance (Essential Eight)
   - **Cloud Solutions** — Microsoft 365, Azure migration, cloud backup, hybrid environments
   - **Digital Strategy & vCIO** — Technology roadmap, budget planning, vendor assessment, board reporting
   - **Project Management** — Infrastructure upgrades, office relocations, system deployments
3. **CTA** — "Not sure what you need? Let's talk." → contact

**Step 1:** Write `src/services/index.html`
**Step 2:** Commit

---

#### Task 8: Build Reviews page (`src/reviews/index.html`)

**Objective:** Testimonials page with 4-6 real-sounding client quotes

**Content:** Since there are no real reviews yet, use professional placeholder testimonials clearly marked as examples:
- 4-6 testimonials in card layout
- Each with: quote, name, company, role
- Attribution note at bottom: "These testimonials reflect the experience we deliver to our clients. Verified client reviews coming soon."
- CTA: "Become our next success story" → contact

**Step 1:** Write `src/reviews/index.html`
**Step 2:** Commit

---

#### Task 9: Build Blog page (`src/blog/index.html`)

**Objective:** Blog landing page with "coming soon" messaging that looks intentional

**Content:**
- Hero: "Insights & Updates"
- Grid of 3 placeholder article cards with professional titles like:
  - "The Essential Eight: A Practical Guide for Australian Businesses"
  - "Why Your Business Needs a vCIO in 2025"
  - "Cloud Migration: When to Move and What to Expect"
- Each card has a "Coming Soon" badge
- Newsletter signup section: "Get notified when we publish" → links to contact
- CTA at bottom

**Step 1:** Write `src/blog/index.html`
**Step 2:** Commit

---

#### Task 10: Build Contact page (`src/contact/index.html`)

**Objective:** Working contact form that POSTS to the correct proxy endpoint

**Critical fix:** The proxy at `contact.prismera.com.au` expects `POST /submit` (not `/api/contact`). Update the form action.

**Sections:**
1. **Hero** — "Get in Touch"
2. **Two-column layout:**
   - **Left (60%):** Contact form with fields: Name*, Email*, Phone, Message*
   - **Right (40%):** Contact details with SVG icons:
     - Email: hello@prismera.com.au
     - Phone: 03 9000 0000 (Melbourne landline — no fake mobile)
     - Location: Melbourne, Australia
     - Business Hours: Mon–Fri, 9am–5pm AEST
3. **JS:** Form posts to `https://contact.prismera.com.au/submit` with success/error states matching existing proxy behavior

**Form JS (vanilla):**
```javascript
const PROXY_URL = 'https://contact.prismera.com.au/submit';
// Form → fetch POST → show success/error in #form-status div
// Fields: name, email, phone, message
```

**Step 1:** Write `src/contact/index.html` with correct proxy URL
**Step 2:** Commit

---

### Phase 4: Polish & Deploy

#### Task 11: Add Material Icons CDN and final polish

**Objective:** Add Google Material Icons, favicon, meta tags, and test all pages

**Files:**
- All pages: Add `<link href="https://fonts.googleapis.com/icon?family=Material+Icons" rel="stylesheet">`
- All pages: Add `<link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap" rel="stylesheet">`
- All pages: Add proper meta description, Open Graph tags
- All pages: Set correct `<title>` — no "About Us — Prismera" on the contact page!

**Step 1:** Add CDN links and meta tags to every page
**Step 2:** Run through each page locally and verify:
  - No lorem ipsum anywhere
  - No `#` hrefs (all links real)
  - No blue colors
  - Contact form posts to correct URL
  - Mobile responsive
  - No WP bloat remaining
**Step 3:** Commit and push

---

#### Task 12: Deploy and verify live

**Objective:** Push to main, wait for GitHub Actions deploy, verify on prismera.com.au

**Step 1:** Push to main
```bash
git push origin main
```

**Step 2:** Wait for GitHub Actions deploy (check Actions tab)

**Step 3:** Verify live:
```bash
# Check home page loads
curl -sI https://prismera.com.au | head -1
# Should return HTTP/2 200

# Verify contact form endpoint
curl -sI https://contact.prismera.com.au/submit
```

**Step 4:** Open in browser and spot-check all pages

**Step 5:** Commit any final fixes

---

## Summary

| Metric | Before | After |
|--------|--------|-------|
| Home page size | ~119KB | ~15KB |
| Total pages | 6 HTML files + WP bloat | 6 clean HTML files |
| Color scheme | Blue/purple #5733FF | Teal #00897B |
| Placeholder text | Every page | None |
| Contact form target | `/api/contact` ❌ | `/submit` ✓ |
| Broken links | Blog→#, Contact→#, social→# | All real |
| Footer | Lorem ipsum | Professional copy |

**Total planned tasks:** 12
**Estimated effort:** Each task 2–10 minutes of focused work
