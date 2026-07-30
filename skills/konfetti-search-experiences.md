---
name: konfetti-search-experiences
description: Search and browse konfetti's catalog of 7,600+ bookable experiences, workshops and courses across Germany and Austria — filtering by category, supplier, type and availability, and reading full experience detail including pricing, ratings and the next available date.
api: konfetti:store
base_url: https://api.gokonfetti.com/v1
generated: '2026-07-19'
method: generated
source: openapi/konfetti-store-openapi.yml
operations:
  - listStoreCategories
  - listCategories
  - listEvents
  - getEvent
  - listSimilarEvents
---

# Search konfetti experiences

konfetti runs a marketplace of bookable experiences — cooking classes, pottery
workshops, cocktail courses, tastings, boat tours, DIY kits — in Germany and
Austria. The catalog is readable without any credentials.

**Before you start, read this.** konfetti publishes no developer portal, no API
reference and no terms of use covering programmatic access. This is an
undocumented internal interface that may change without notice. There is no
published rate limit, so throttle conservatively. Do not build a production
dependency on it without asking konfetti first (hallo@gokonfetti.com).

## Step 1 — get the category vocabulary

Call `listStoreCategories` (`GET /v1/store/categories`) first. You need the
category **slugs** (`kochkurs`, `cocktailkurs`, `tasting`, `toepferkurs`) to
build filters in the next step. The response also carries aggregate
`avg_review` and `total_reviews` per category.

If you need landing-page copy — SEO titles, descriptions, the parent/child
hierarchy — call `listCategories` (`GET /v1/categories`) instead; it is the
same taxonomy with content attached and a `parent_id` field.

## Step 2 — search the catalog

Call `listEvents` (`GET /v1/store/events`). The filter grammar is
l5-repository style, not standard OpenAPI query params:

- `search` — semicolon-delimited `field:value` pairs, e.g.
  `search=hasEvent:true;type:ONLINE;category:kochkurs`. Observed fields:
  `supplier_id`, `hasEvent`, `type`, `category`.
- `searchFields` — per-field comparison operator, e.g. `supplier_id:!=` to negate.
- `orderBy` — comma-separated: `created_at`, `distance`, `has_events`,
  `has_promotion`, `trending_90_days`.
- `sortedBy` — `asc` or `desc`.
- `include` — comma-separated relations: `categories`, `supplier`, `address`,
  `reviews`, and the dot-path `address.locality`.
- `page` / `per_page` (default 10) / `limit` (`limit=-1` for unpaginated).

Send `include` and `orderBy` as bare comma-separated lists — do not
URL-encode the commas; the first-party client sends them unencoded.

## Step 3 — page through results

Read `meta.pagination`: `total`, `count`, `per_page`, `current_page`,
`total_pages`, and `links.next` / `links.previous`. The links are absolute and
carry your other query parameters forward, so follow `links.next` rather than
incrementing `page` yourself. There were 7,605 experiences on 2026-07-19, so
plan for ~761 pages at the default size.

## Step 4 — read one experience in full

Call `getEvent` (`GET /v1/store/events/{permalink}`).

**The lookup key is the permalink, not the id.** Use the `permalink` field from
the search result (e.g. `cocktailkurs-in-muenchen-0w1dlw`). Passing the bare
six-character id returns 404 with
`{"message":"The requested Resource was not found.","errors":[]}`.

The detail response adds a full HTML `description` body that the list response
omits.

## Step 5 — find related experiences

Call `listSimilarEvents` (`GET /v1/store/events/{permalink}/similar`). To
mirror what konfetti's own site does, pass
`orderBy=distance&sortedBy=asc&search=supplier_id:<id>;hasEvent:true&searchFields=supplier_id:!=`
— that finds nearby experiences excluding the same supplier.

## Reading the data correctly

- **Money is an object, and the amount is a string of minor units.**
  `{"amount":"2900","currency":"EUR","formatted":"29,00 €"}` means €29.00.
  Parse `amount` as an integer and divide by 100. Never treat it as a float.
- **Types are carried in an `object` field**, not an id prefix. Ids are plain
  six-character lowercase alphanumerics (`xyn62z`) with no type information.
- **Included relations nest under their own `data` key**:
  `supplier: {data: {...}}`, `categories: {data: [...]}`.
- **`next_instance`** is the next scheduled date, with `status`,
  `available_tickets_quantity` and `date_type` (`PUBLIC` or `PRIVATE`).
- **Locale matters.** Send `Accept-Language` to control the response language.
  Formatted prices, `readable_created_at` ("vor 1 Jahr") and validation
  messages all come back translated. Responses carry `content-language`
  (observed `de_DE` — note the underscore form, not BCP 47).

## Error handling

konfetti does **not** use RFC 9457 problem details. See
`errors/konfetti-problem-types.yml`. What you will actually see:

- `404` → `{"message":"The requested Resource was not found.","errors":[]}` —
  usually means you passed an id where a permalink was expected.
- `405` → an HTML page, not JSON. You used the wrong verb.
- `422` → `{"message":..., "code":422, "reason":"requestValidation",
  "errors":{"field":["message"]}}`. Key off the field names; the messages are
  localized and not machine-stable.
- `500` → an HTML Symfony error page. Check `content-type` before parsing any
  error body.
- `302` → you hit a protected endpoint without a token. **Disable
  redirect-following**, or you will parse an HTML login page as JSON.

## What you cannot do here

Nothing on this skill writes. Pricing, cart validation, coupon validation,
orders and profile all require a Bearer token from
`POST /v1/oauth/token`, and konfetti publishes no client-registration path, so
third parties cannot obtain one. The lead-capture endpoints (date requests,
private event requests, newsletter) create real records in konfetti's systems
and are not idempotent — do not call them from an agent.
