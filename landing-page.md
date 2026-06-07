# ZygoRIde Landing Page & Legal Documents

## Tech Stack

Pure HTML5 / CSS3 / Vanilla JavaScript — no framework, no build step.

| Asset | Detail |
|---|---|
| Fonts | Bricolage Grotesque (headings) + Epilogue (body) via Google Fonts |
| Stylesheet | `assets/css/styles.css` |
| Scripts | `assets/js/main.js` |
| Contact form | Formspree (endpoint ID `xdawobkq`) |
| SEO | Schema.org `MobileApplication` structured data, Open Graph, Twitter Card |
| PWA | `site.webmanifest` |
| Sitemap | `sitemap.xml` |

Dark/light theme toggle is implemented in `main.js` (persists to `localStorage`). Smooth-scroll + intersection-observer reveal animations are applied to elements with `.reveal` class.

---

## Page Structure — `index.html`

### Nav

Sticky top navigation with logo, four anchor links (Features, How It Works, Preview, Support), a dark/light theme toggle button, and a "Download App" CTA. Includes a hamburger menu for mobile. All links are in-page anchors.

### Hero (`#home`)

Full-viewport hero section. Left column: eyebrow tag "Now Live · India", headline "Your ride. Your people. Your commute.", tagline, app store buttons (currently "Coming Soon" badges, non-clickable), and three stats chips (20+ Cities, Two-way Rating ★, Free to Download). Right column: three stacked phone frame mockups showing `scheduled.png`, `home.png`, and `results.png` with floating notification cards (animated).

### Trust Bar

Horizontally scrolling marquee strip (CSS infinite scroll animation, duplicated for seamless loop). Items: ID Verified Users, Live Ride Tracking, In-App Messaging, Two-Way Ratings, India Launch.

### Features (`#features`)

Bento-grid card layout. Seven feature cards:

| Card | Title | Key points |
|---|---|---|
| A (wide) | Smart Route Matching | H3 spatial algorithm, partial address, city-level filter, date range |
| B | Live Ride Tracking | Real-time GPS, ETA sharing |
| C | Verified Users | Government ID check for every user |
| D | In-App Chat | Coordinate pickup without sharing phone numbers |
| E (full-width) | Conflict-Free Scheduling | Prevents double-bookings; shows sample weekly schedule |
| F (wide) | Request a Ride | Post request; auto-notified when a provider posts a matching ride |
| G | Providers Discover Riders | Drivers browse passenger requests; distance filter; direct offer |
| H | Smart Notifications | Match alerts, booking confirmations, ride reminders |
| I | Flexible Payments – Zero Commission | Peer-to-peer cash/UPI; no platform fee |

### Pricing Callout

Between Features and How It Works. Highlights: Free to download, typical fare ₹50–₹200, zero commission, peer-to-peer payment.

### How It Works (`#how-it-works`)

Tabbed section with two tabs: **Rider** and **Ride Provider**. Each has 4 illustrated steps.

**Rider tab steps:**
1. Create profile + upload ID
2. Search by route/date OR post a Ride Request (get notified on auto-match)
3. Book + chat with driver to confirm pickup
4. Ride with live map; rate afterwards; see history in Rides tab

**Ride Provider tab steps:**
1. Create profile + add vehicle + upload ID
2. Post a ride OR browse passenger requests and send a direct offer
3. Confirm bookings; chat with passengers
4. Start ride with OTP check; complete and get rated; see history in Rides tab

### App Preview (`#preview`)

Screenshot gallery showing 5 app screens (search, results, ride details, driver dashboard, profile). Uses `<picture>` tags with WebP `<source>` + PNG fallback; `loading="lazy"`.

### Download CTA

Secondary call to action section with store buttons (same Coming Soon state as hero). Links to `#download`.

### Support (`#support`)

Contact form powered by Formspree (`action="https://formspree.io/f/xdawobkq"`). Fields: name, email, subject, message. On submission, shows inline success/error feedback via `main.js` (no page reload). Social media links in footer (currently `href="#"` placeholders).

### Footer

Logo, nav links, legal links (Terms, Privacy), social icons (Instagram, Facebook, X, LinkedIn — all placeholder `#`), copyright.

---

## Terms of Use — `terms.html`

22 sections governing use of the platform.

| # | Section | Summary |
|---|---|---|
| 1 | About ZygoRIde | Peer-to-peer cost-sharing platform, not a licensed transport company. Aggregator clause: operates for private carpooling under the Motor Vehicles Act, not as a commercial aggregator. |
| 2 | Acceptance | Using the app = accepting Terms. |
| 3 | Eligibility | Must be 18+, valid driving licence for providers. |
| 4 | Account Registration | Accurate info required; user responsible for account security. |
| 5 | Identity Verification | Government ID required. No criminal background checks performed (disclaimer included). Vehicle compliance (PUC + insurance) required from providers. |
| 6 | Ride Provider Obligations | Must have valid DL, RC, insurance, PUC; no commercial taxis; must honour confirmed bookings. |
| 7 | Rider Obligations | Respectful behaviour; timely arrival; honest ratings. |
| 8 | Payments | Peer-to-peer cash/UPI; ZygoRIde never holds funds. Refund policy: ZygoRIde charges no fee, so no refund applies from ZygoRIde. |
| 9 | Safety Features Disclaimer | SOS button is best-effort; not a guaranteed emergency service. Always call 112. |
| 10 | Tax Obligations | Ride Providers solely responsible for income tax and GST compliance. ZygoRIde does not deduct TDS. |
| 11 | Prohibited Conduct | Commercial use, fraud, harassment, illegal activity. |
| 12 | Platform Liability | ZygoRIde is a marketplace; not liable for ride outcomes. Liability cap: ₹500 or fare paid (whichever lower). |
| 13 | Insurance Disclaimer | ZygoRIde provides no insurance. Users ride at own risk. |
| 14 | Dispute Resolution | Disputes between riders/providers are their own. Courts: Bengaluru, Karnataka, India. |
| 15 | Intellectual Property | ZygoRIde owns all platform IP. |
| 16 | Privacy | Governed by Privacy Policy. |
| 17 | Severability | Invalid provisions are modified minimally; remainder stays in force. |
| 18 | Force Majeure | ZygoRIde not liable for outages beyond its control (includes Firebase/GCP). |
| 19 | Entire Agreement | Terms + Privacy Policy supersede all prior agreements. |
| 20 | Termination | ZygoRIde may suspend accounts; users may delete accounts. |
| 21 | Governing Law | Republic of India, IT Act 2000. |
| 22 | Changes to Terms | ZygoRIde may update; continued use = acceptance. |

---

## Privacy Policy — `privacy.html`

11 sections covering data practices under Indian law.

| # | Section | Summary |
|---|---|---|
| 1 | Information We Collect | Name, email, phone, ID documents (deleted within 7 days; Aadhaar numbers never stored), vehicle info, GPS (active ride only — no background GPS outside rides), chat messages, device tokens. |
| 2 | How We Use Information | Ride matching, safety, customer support, notifications, platform improvement. |
| 3 | Legal Basis | Consent + contractual necessity (IT Act 2000, DPDP Act 2023). Explicit consent for Sensitive Personal Data. |
| 4 | Data Sharing | Only with ride counterparties (name, phone for coordination), Firebase/Google Cloud (processor), Formspree (contact form), Google Fonts (homepage). No sale of data. |
| 5 | Data Retention | ID docs: 7 days. Chat messages: 90 days. Ride data: 12 months. Account data: until deletion request. |
| 6 | User Rights | Access, correction, deletion, withdrawal of consent. Contact: zygoride.feedback@gmail.com. |
| 7 | Grievance Officer | "ZygoRIde Operations" (placeholder — real name required by IT Rules 2011 Rule 5(9) before publishing). Email: zygoride.feedback@gmail.com. 30-day response SLA. |
| 8 | Children's Privacy | App not directed at under-18s; no knowing collection. |
| 9 | Security | HTTPS, Firebase security rules, encrypted ID storage. CERT-In breach reporting within required timeframe. |
| 10 | Data Storage Location | Firebase Google Cloud, asia-south1 region (Mumbai). |
| 11 | Changes to Policy | Material changes notified via app; continued use = acceptance. |

---

## Contact Form

**Provider:** Formspree  
**Endpoint:** `https://formspree.io/f/xdawobkq`  
**Fields:** `name`, `email`, `subject`, `message`  
**Behaviour:** `main.js` intercepts `submit`, posts JSON via `fetch`, shows inline success/error message in a `#form-status` element without page reload.

---

## Outstanding TODOs from `improvements.md`

| # | Item | Priority | Status |
|---|---|---|---|
| 11 | Grievance Officer real name in `privacy.html` §7 | 🟡 Medium | ⚠ Manual — add legal name before publish |
| 13 | Convert screenshots to WebP (use squoosh.app) | 🔵 Launch | ❌ Todo |
| 14 | Update App Store / Play links when live | 🔵 Launch | ❌ Todo |
| 15 | Create favicon files (32px, 16px, apple-touch, OG image, PWA icons) | 🔵 Launch | ❌ Todo |
| 16 | Add real social media URLs to footer | 🔵 Launch | ❌ Todo |
| 17 | Verify Formspree endpoint `xdawobkq` is active | 🔵 Launch | ❌ Todo |
| 18 | Update support email to domain address once ready | 🔵 Launch | ❌ Todo |

All 🔴 Critical and 🟠 High legal items (Motor Vehicles Act aggregator clause, Aadhaar prohibition, CERT-In breach reporting, background check disclaimer, vehicle compliance, SOS disclaimer, tax disclaimer, severability/force majeure/entire agreement, refund policy) have been completed.
