# SVCRO — Official Website

Maximalist streetwear brand site. Pure HTML/CSS/JS — no frameworks, no build step, deploy-ready for AWS S3 + CloudFront.

---

## Files

```
SVCRO_website/
├── index.html    — Main website (5 pages, client-side routed)
├── admin.html    — Admin dashboard (login-protected)
├── DEPLOY.md     — Step-by-step AWS deployment guide
└── README.md     — This file
```

---

## Running Locally

No build step required. Open directly in a browser:

```bash
open index.html
open admin.html
```

Or serve with any static file server to avoid potential CORS issues with future fetch calls:

```bash
# Python
python3 -m http.server 8080

# Node (npx)
npx serve .
```

Then visit `http://localhost:8080`.

---

## Main Site — `index.html`

Five pages rendered client-side via `showPage()` — no page reloads.

| Page | Route | Key Features |
|------|-------|-------------|
| Home | `showPage('home')` | Animated hero, ticker, marquee, drops grid, statement block, collections preview |
| Shop | `showPage('shop')` | 12-product grid, category filter, quick-view hover |
| Collections | `showPage('collections')` | Editorial full-height alternating layout, 4 collections |
| About | `showPage('about')` | Brand story, stats grid, values cards, team section |
| Media | `showPage('media')` | Gallery / Videos / Press tab system |

### Design System

| Token | Value |
|-------|-------|
| `--black` | `#0a0a0a` |
| `--white` | `#f5f0e8` |
| `--yellow` | `#FFE135` |
| `--orange` | `#FF4D00` |
| `--red` | `#E8003D` |
| `--cyan` | `#00E5FF` |
| `--purple` | `#9B00FF` |
| `--green` | `#00FF88` |
| `--pink` | `#FF2D78` |

**Fonts** — Bebas Neue (display), Space Mono (body), Syne (UI labels) — loaded from Google Fonts.

**Custom cursor** — 16px yellow dot + 40px ring trail, `mix-blend-mode: difference`.

---

## Admin Dashboard — `admin.html`

**Login:** `admin` / `svcro2025`

| Panel | Purpose |
|-------|---------|
| Dashboard | Stats overview, recent products, activity feed |
| Products | Full CRUD — add, edit, delete all 12 products |
| Collections | Manage the 4 collections with status chips |
| Media | Upload UI (drag & drop), thumbnail grid |
| Page Content | Accordion form editors for all copy-managed sections |
| Press | Add/remove press features (outlet, quote, date) |
| Settings | Brand config, color pickers, AWS fields, integrations |

> **Note:** All data is in-memory JS arrays. See `DEPLOY.md` §8 for the path to real persistence via DynamoDB + Lambda, or a simpler S3 JSON file approach.

---

## Deployment

See **[DEPLOY.md](DEPLOY.md)** for the full guide. Quick summary:

```bash
BUCKET=svcro-website-prod
DIST_ID=YOUR_CLOUDFRONT_ID

aws s3 cp index.html s3://$BUCKET/ --content-type "text/html"
aws s3 cp admin.html s3://$BUCKET/ --content-type "text/html"
aws cloudfront create-invalidation --distribution-id $DIST_ID --paths "/*"
```

Estimated AWS cost: **~$1.50/month** at moderate traffic.

---

## Extending the Site

| Task | Where to look |
|------|--------------|
| Add a product | `admin.html` → Products panel, or edit the `products` array directly in `index.html` |
| Change hero copy | `index.html` hero section, or `admin.html` → Page Content → Homepage Hero |
| Add a new page | Add a `<div id="page-X" class="page">` block and a `showPage('X')` nav link |
| Wire newsletter | Replace `handleNewsletter()` stub in `index.html` with a Klaviyo API call |
| Wire analytics | Add `gtag.js` snippet to `<head>` with your GA4 measurement ID |
| Add real images | Replace `linear-gradient` placeholder divs with `<img>` tags or `background-image` URLs |
| Persist admin data | See `DEPLOY.md` §8 — S3 JSON or DynamoDB + API Gateway |

---

## Instagram

[@svcroofficial](https://www.instagram.com/svcroofficial/)
