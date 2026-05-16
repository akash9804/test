# Hardcoded SEO JSON-LD inventory

This document lists **every JSON-LD (Schema.org) block** that is still defined or driven from **code**, so the same shapes can be recreated in **Admin → Content → SEO · JSON-LD schemas** (`page_json_ld_schemas`).

Related code:

| Area | Path |
|------|------|
| Schema builders | `src/lib/seo/schemaBuilders.js` |
| Merge code + DB | `src/lib/seo/resolvePageJsonLd.js` |
| Route keys | `src/lib/seo/pageSeoRoutes.js` |
| Course defaults | `src/lib/seo/coursePageJsonLdDefaults.js` |
| Admin UI | `src/components/AdminCms/PageJsonLdEditor.jsx` |
| DB table | `admin/db/page_json_ld.sql` |

---

## How rendering works today

```
flowchart TB
  subgraph layout [Root layout — every page]
    H1["Hardcoded: org-schema + website-schema"]
    H2["DB: global-head / global-footer"]
  end
  subgraph page [Per-route page component]
    D["Code defaults by script_id"]
    DB["DB rows for page_route_key"]
    M["mergePageJsonLdPlans — DB wins on same script_id"]
    R["resolveJsonLdFromPlan + runtime context"]
  end
  H1 --> HTML
  H2 --> HTML
  D --> M --> R --> HTML
  DB --> M
```

1. **`layout.js`** always injects **Organization** and **WebSite** scripts (`id="org-schema"`, `id="website-schema"`). These are **not** in `page_json_ld_schemas` unless you duplicate them under route `global`.
2. Each page calls `buildPageJsonLdForRoute({ routeKey, defaults, context })`.
3. **Code `defaults`** register expected `script_id`s and `schema_kind`s.
4. **DB rows** with the **same `script_id`** override or disable (`enabled: false`) a code default.
5. Kinds `breadcrumb`, `faq`, `course`, `organization`, `website` are built from **`context`** (live CMS/catalog data) unless `config` in DB overrides items.

**After CMS publish:** clear public cache / revalidate so Next.js picks up DB changes.

---

## Tier 1 — Hardcoded in `layout.js` (all pages)

These run on **every** public page and are **not** managed by per-route `buildPageJsonLdForRoute` today.

| HTML `id` | `@type` | Source file | Move to admin? |
|-----------|---------|-------------|----------------|
| `org-schema` | `Organization` | `src/app/layout.js` | Yes → route `global`, placement `global-head`, kind `organization` **or** `raw` |
| `website-schema` | `WebSite` | `src/app/layout.js` | Yes → route `global`, placement `global-head`, kind `website` **or** `raw` |

### Organization (current hardcoded values)

Use env `NEXT_PUBLIC_SITE_URL` for production URLs (example below uses `https://www.aaftonline.com`).

| Field | Value |
|-------|--------|
| `name` | `AAFT Online` |
| `url` | `{SITE_URL}` |
| `logo` | `{SITE_URL}/images/aaft-logo.png` (via `publicAssetUrl`) |
| `description` | `AAFT Online offers industry-led creative and professional online diploma and certificate programs.` |
| `sameAs` | `https://www.instagram.com/aaftonline/` |
| | `https://www.youtube.com/@AAFTONLINE` |
| | `https://www.facebook.com/AAFTONLINE` |
| | `https://www.linkedin.com/company/aaft-online/` |

**Admin entry (kind `organization`):** route `global`, script_id `org-schema`, placement `global-head`, config:

```json
{
  "organization": {
    "siteUrl": "https://www.aaftonline.com",
    "logoUrl": "https://www.aaftonline.com/images/aaft-logo.png",
    "sameAs": [
      "https://www.instagram.com/aaftonline/",
      "https://www.youtube.com/@AAFTONLINE",
      "https://www.facebook.com/AAFTONLINE",
      "https://www.linkedin.com/company/aaft-online/"
    ],
    "description": "AAFT Online offers industry-led creative and professional online diploma and certificate programs."
  }
}
```

**Or kind `raw`** with full JSON-LD object in `json_ld` (same fields as `generateOrganizationSchema` output).

### WebSite (current hardcoded values)

| Field | Value |
|-------|--------|
| `@type` | `WebSite` |
| `url` | `{SITE_URL}` |

**Admin entry (kind `website`):** route `global`, script_id `website-schema`, placement `global-head`, config:

```json
{ "siteUrl": "https://www.aaftonline.com" }
```

**Note:** To *remove* duplicate scripts after migrating to DB, you must later delete or gate the hardcoded blocks in `layout.js` (lines with `ORGANIZATION_SCHEMA` / `WEBSITE_SCHEMA`).

---

## Tier 2 — Per-page code defaults (merge with DB)

Admin route picker uses `page_route_key` from `src/lib/seo/pageSeoRoutes.js`.

### Static routes

| `page_route_key` | Page path | Code source | Default `script_id`s | `schema_kind` | Runtime data source |
|------------------|-----------|-------------|----------------------|-----------------|------------------------|
| `home` | `/` | `src/app/page.js` | `home-faq-schema` (if FAQs exist) | `faq` | `sections.faq` (home CMS) |
| `about` | `/about-us` | `src/app/about-us/page.jsx` | `about-breadcrumb-schema` | `breadcrumb` | Home → About us |
| | | | `about-faq-schema` (if items) | `faq` | About page FAQ CMS |
| `explore-course` | `/explore-course` | `src/app/explore-course/page.jsx` | `explore-course-breadcrumb-schema` | `breadcrumb` | Home → Explore Courses |
| `faqs` | `/faqs` | `src/app/faqs/page.jsx` | `faqs-breadcrumb-schema` | `breadcrumb` | Home → FAQs |
| | | | `faqs-faq-schema` (if entries) | `faq` | `faq_page_entries` table |
| `diploma-courses` | `/diploma-courses` | `src/app/diploma-courses/page.jsx` | `diploma-courses-breadcrumb-schema` | `breadcrumb` | Home → Diploma Courses |
| `press-release` | `/press-release` | `src/app/press-release/page.jsx` | `press-release-breadcrumb-schema` | `breadcrumb` | Home → Press Release |
| `global` | `*` (all pages) | `src/app/layout.js` | _(none in code)_ | — | Optional extra head/footer via `global-head` / `global-footer` |

#### Breadcrumb items to enter in admin (`config.items`)

Use absolute URLs with your production origin.

**`explore-course`**

```json
{
  "items": [
    { "name": "Home", "url": "https://www.aaftonline.com" },
    { "name": "Explore Courses", "url": "https://www.aaftonline.com/explore-course" }
  ]
}
```

**`about`**

```json
{
  "items": [
    { "name": "Home", "url": "https://www.aaftonline.com" },
    { "name": "About us", "url": "https://www.aaftonline.com/about-us" }
  ]
}
```

**`faqs`**

```json
{
  "items": [
    { "name": "Home", "url": "https://www.aaftonline.com" },
    { "name": "FAQs", "url": "https://www.aaftonline.com/faqs" }
  ]
}
```

**`diploma-courses`**

```json
{
  "items": [
    { "name": "Home", "url": "https://www.aaftonline.com" },
    { "name": "Diploma Courses", "url": "https://www.aaftonline.com/diploma-courses" }
  ]
}
```

**`press-release`**

```json
{
  "items": [
    { "name": "Home", "url": "https://www.aaftonline.com" },
    { "name": "Press Release", "url": "https://www.aaftonline.com/press-release" }
  ]
}
```

#### FAQ schemas (`config.items` or live context)

For `home`, `about`, `faqs`: if you add a DB row with the **same `script_id`** as the code default and kind `faq`:

- Leave `config.items` **empty** → page still passes `faqItems` in **context** (recommended; stays in sync with CMS).
- Or set `config.items` explicitly:

```json
{
  "items": [
    { "q": "Question text?", "a": "Answer text." }
  ]
}
```

Aliases accepted: `question`/`answer` instead of `q`/`a`.

---

### Course routes (`course:{cmsCoursePageKey}`)

Each live course page uses `CourseMarketingPage.jsx` with three code defaults per course (`coursePageJsonLdDefaults`).

| `page_route_key` | Public path | `schemaIdPrefix` (script_id prefix) | Default script IDs |
|------------------|-------------|-------------------------------------|--------------------|
| `course:fashion` | `/diploma-in-fashion-design` | `fashion` | `fashion-breadcrumb-schema`, `fashion-course-schema`, `fashion-faq-schema` |
| `course:nutrition` | `/diploma-in-nutrition` | `nutrition` | `nutrition-*` |
| `course:music` | `/diploma-in-music-production-online` | `music` | `music-*` |
| `course:interior` | `/diploma-in-interior-design` | `interior` | `interior-*` |
| `course:animation` | `/diploma-in-animation-and-vfx` | `animation` | `animation-*` |
| `course:jewelry` | `/diploma-in-jewelry-design` | `jewelry` | `jewelry-*` |
| `course:event` | `/diploma-in-event-management` | `event` | `event-*` |
| `course:hospital` | `/diploma-in-hospital-management` | `hospital` | `hospital-*` |
| `course:travel-tourism` | `/diploma-in-travel-and-tourism` | `travel-tourism` | `travel-tourism-*` |
| `course:culinary-arts` | `/diploma-in-culinary-arts` | `culinary-arts` | `culinary-arts-*` |
| `course:real-estate-management` | `/diploma-in-real-estate` | `real-estate` | `real-estate-*` |
| `course:psychology` | `/diploma-in-psychology` | `psychology` | `psychology-*` |
| `course:hotel-management` | `/diploma-in-hotel-management` | `hotel-management` | `hotel-management-*` |
| `course:fashion-styling` | `/diploma-in-fashion-styling` | `fashion-styling` | `fashion-styling-*` |
| `course:ayurvedic-wellness` | `/diploma-in-ayurvedic-wellness` | `ayurvedic-wellness` | `ayurvedic-wellness-*` |
| `course:screenwriting` | `/diploma-in-screenwriting` | `screenwriting` | `screenwriting-*` |
| `course:naturopathy` | `/diploma-in-naturopathy` | `naturopathy` | `naturopathy-*` |
| `course:makeup` | `/diploma-in-makeup` | `makeup` | `makeup-*` |
| `course:podcasting-media` | `/diploma-in-Podcasting` | `podcasting` | `podcasting-*` |
| `course:ad-film-making` | `/ad-film-making-online` | `ad-film-making` | `ad-film-making-*` |
| `course:certificate-advertising-pr` | `/advertising-pr-corporate-communication` | `advertising-pr` | `advertising-pr-*` |
| `course:journalism` | `/the-ultimate-journalism` | `journalism` | `journalism-*` |
| `course:radio-jockey` | `/radio-jockey` | `radio-jockey` | `radio-jockey-*` |
| `course:online-dj-course` | `/certification-in-djing` | `certification-djing` | `certification-djing-*` |
| `course:photography-comprehensive` | `/professional-photography` | `professional-photography` | `professional-photography-*` |

**Admin route key** uses catalog `cms_id` (= `cmsCoursePageKey`), e.g. `course:fashion`, not always the same as `schemaIdPrefix`.

#### Course breadcrumb (code default pattern)

All courses use the same **structure** (middle crumb label is "Diploma Courses", URL is **explore-course**, not diploma-courses listing):

```json
{
  "items": [
    { "name": "Home", "url": "https://www.aaftonline.com" },
    { "name": "Diploma Courses", "url": "https://www.aaftonline.com/explore-course" },
    { "name": "{Program display name}", "url": "https://www.aaftonline.com{canonicalPath}" }
  ]
}
```

Override per course in admin with kind `breadcrumb` + matching `script_id` (e.g. `fashion-breadcrumb-schema`).

#### Course schema (`schema_kind: course`)

Built by `generateCourseSchema()` from **runtime context** in `CourseMarketingPage.jsx` (catalog + CMS). Fields:

| Field | Source |
|-------|--------|
| `name` | Brochure card title or hero program name |
| `description` | Card subtitle or `courseSchemaDescriptionFallback` from registry |
| `url` | Canonical course URL |
| `image` | Hero image or default OG image |
| `aggregateRating.ratingValue` | Card `rating` (default `4.8`) |
| `aggregateRating.ratingCount` | Card `reviewCount` (default `840`) |
| `offers` | `{ @type: Offer, category: Paid, priceCurrency: INR }` |
| `totalHistoricalEnrollment` | Card `enrolled` (default `2400`) |
| `about` | Basic-info job titles or card `jobs` |
| `teaches` | Basic-info learn bullets or card `learn` |
| `educationalCredentialAwarded` | Diploma credential (name = program, url = page) |
| `coursePrerequisites` | Fixed string: no specific prerequisites |
| `hasCourseInstance` | Online, workload `P1Y` |
| `publisher` / `provider` | AAFT Online (fixed in builder) |

**Recommendation:** Keep kind `course` in code (dynamic). Use DB only to **disable** (`enabled: false`) or add **extra** `raw` blocks. For a fully static course JSON-LD in admin, use kind `raw` and paste full JSON (will **not** auto-update when catalog changes).

#### Course FAQ (`{prefix}-faq-schema`)

- Source: course page **FAQ block** (`sections.coursePages[cmsKey].faq`), not site-wide home FAQ.
- Same Q/A as visible FAQ section when CMS is updated.
- Admin: kind `faq`, empty `config.items` → still uses live context unless you override items.

---

## Tier 3 — Schema kind reference (admin)

| `schema_kind` | Generated `@type` | `config` keys | `json_ld` |
|---------------|-------------------|---------------|-----------|
| `raw` | _(any)_ | optional `jsonLd` | **Required** full object |
| `breadcrumb` | `BreadcrumbList` | `items[]` with `name` + `url` | — |
| `faq` | `FAQPage` | `items[]` with `q`/`a` (or question/answer) | — |
| `course` | `Course` | — | — (uses page `context.course`) |
| `organization` | `Organization` | `organization` object | — |
| `website` | `WebSite` | `siteUrl` | — |

**Placements**

| `placement` | Where rendered |
|-------------|----------------|
| `page` | That route's page component only |
| `global-head` | Root layout (after hardcoded org/website) |
| `global-footer` | Root layout end of `<body>` |

---

## Migrating a code default into the backend

1. Open **Admin → Content → SEO · JSON-LD schemas**.
2. Select the **`page_route_key`** (e.g. `explore-course`, `course:fashion`, `global`).
3. Add a row:
   - **Script ID** — must match code default exactly to override (e.g. `explore-course-breadcrumb-schema`).
   - **Label** — admin-only note.
   - **Kind** — `breadcrumb` | `faq` | `raw` | etc.
   - **Placement** — usually `page` (or `global-head` for org/website).
   - **Enabled** — `false` to turn off a code default without deleting.
   - **Config / JSON-LD** — per kind above.
4. Save → clear CDN / revalidate site cache.

To **stop using code defaults entirely** for a route: add DB rows for every `script_id` you need, then (later) remove the `defaults` array from that page's `buildPageJsonLdForRoute` call.

---

## Not JSON-LD (separate SEO)

These are **`<meta>` / Open Graph** via `createPageMetadata()` or `layout.js` `metadata` — **not** in `page_json_ld_schemas`:

| Page | File | Notes |
|------|------|--------|
| Home | `src/app/page.js` | title, description, canonical `/` |
| Explore | `src/app/explore-course/page.jsx` | |
| About | `src/app/about-us/page.jsx` | dynamic from CMS SEO band when present |
| FAQs | `src/app/faqs/page.jsx` | |
| Diploma listing | `src/app/diploma-courses/page.jsx` | |
| Press release | `src/app/press-release/page.jsx` | |
| Each course | `src/app/{canonical}/page.jsx` + registry `defaultTitle` / `defaultDescription` | |
| Legal | `privacy-policy`, `terms-conditions` | |
| Root defaults | `src/app/layout.js` | fallback title/description |

Future work could add a `page_seo_meta` table; today only JSON-LD is admin-managed.

---

## Quick checklist

- [ ] Migrate **global** Organization + WebSite to `global` / `global-head` (then remove layout hardcoding when ready).
- [ ] Confirm static breadcrumbs in admin for explore, about, faqs, diploma-courses, press-release (or leave code defaults).
- [ ] Home / about / faqs FAQ schemas — prefer live context; document overrides only if needed.
- [ ] Per-course: decide keep dynamic `course` + `faq` in code vs static `raw` in DB.
- [ ] Verify `script_id` matches table above when overriding.
- [ ] Revalidate production after bulk import.

---

## Example: disable code FAQ on home only

| Field | Value |
|-------|--------|
| `page_route_key` | `home` |
| `script_id` | `home-faq-schema` |
| `schema_kind` | `faq` |
| `enabled` | `false` |
| `placement` | `page` |

Code default is suppressed; add a different `script_id` with kind `raw` if you replace it.
