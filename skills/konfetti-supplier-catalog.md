---
name: konfetti-supplier-catalog
description: Build a picture of a konfetti partner host (supplier) — their full experience catalog, pricing range, review standing and geographic footprint — by pivoting from any single experience to everything that supplier offers on the marketplace.
api: konfetti:store
base_url: https://api.gokonfetti.com/v1
generated: '2026-07-19'
method: generated
source: openapi/konfetti-store-openapi.yml
operations:
  - listEvents
  - getEvent
  - listSimilarEvents
---

# Profile a konfetti supplier

konfetti's marketplace is a two-sided one: partner hosts (suppliers) publish
experiences, and konfetti sells them. This skill assembles a supplier's whole
catalog from the public API, which is useful for competitive research, partner
due diligence, or building a supplier-scoped view.

**Same caveat as every konfetti skill.** This is an undocumented internal API
with no published terms of use, no rate limit contract and no versioning
policy. Throttle conservatively and ask konfetti (hallo@gokonfetti.com) before
depending on it.

## Step 1 — get a supplier id

Suppliers are not directly listable — there is no `/v1/store/suppliers` index.
You pivot from an experience.

Call `listEvents` (`GET /v1/store/events`) with `include=supplier`, then read
`supplier.data.id` off any result. Supplier ids are six-character lowercase
alphanumerics (e.g. `zpe9ow`), same format as every other konfetti id, with the
type carried in `supplier.data.object` = `Supplier`.

The supplier object also gives you `name`, `page_title`, `permalink` and an
HTML `description`.

## Step 2 — pull that supplier's full catalog

Call `listEvents` again, scoped to the supplier:

```
GET /v1/store/events
  ?search=supplier_id:<supplierId>;hasEvent:false
  &include=supplier,categories,address.locality
  &orderBy=created_at
  &limit=-1
```

Notes on that call, all taken from how konfetti's own storefront does it:

- `search` uses semicolon-delimited `field:value` pairs.
- `hasEvent:false` includes listings with no currently scheduled date;
  use `hasEvent:true` to see only bookable-right-now inventory. Running both
  and diffing them tells you how much of the supplier's catalog is dormant.
- `limit=-1` requests an unpaginated result. If you would rather page, drop it
  and follow `meta.pagination.links.next`.

## Step 3 — derive the supplier profile

From the collected experiences, compute:

- **Catalog size** — count of experiences, and the `hasEvent:true` vs
  `hasEvent:false` split.
- **Price range** — from each experience's `min_price` / `max_price` /
  `default_price`. Remember `amount` is a **string of minor units**: `"2900"`
  is €29.00. Parse as integer, divide by 100.
- **Review standing** — `avg_review`, `total_reviews`, and the
  `review_breakdown` histogram (`1_star`..`5_star`) per experience; aggregate
  weighted by `total_reviews`.
- **Format mix** — the `type` field (`NATIONWIDE`, `ONLINE`) and
  `specialization` (`KIT`), plus `is_team_event`, `is_group_pricing`,
  `has_mobile_classes` and `multiple_locations`.
- **Geography** — from `include=address.locality`. Note that `address` comes
  back `null` for `NATIONWIDE` and kit experiences that have no fixed venue —
  handle the null rather than assuming a location.
- **Freshness** — `created_at` and `updated_at`, plus `next_instance.start` for
  how far out they are scheduling.
- **Categories served** — the union of `categories.data[].slug`.

## Step 4 — map the competitive neighbourhood

For a supplier's flagship experience, call `listSimilarEvents`
(`GET /v1/store/events/{permalink}/similar`) with:

```
?orderBy=distance&sortedBy=asc
&search=supplier_id:<supplierId>;hasEvent:true
&searchFields=supplier_id:!=
&include=supplier,categories
```

The `searchFields=supplier_id:!=` negation is what excludes the supplier's own
listings, leaving you the nearest competing suppliers for the same kind of
experience.

## Step 5 — check for a white-label presence

Some partners get a konfetti-hosted subdomain. `GET
/v1/store/suppliers/{id}/subdomain` resolves it — but it returned 404 for the
supplier sampled on 2026-07-19, so treat a 404 as "no white-label configured"
rather than an error.

Partners with embeddable surfaces also appear at
`https://gokonfetti.com/{locale}/widgets/reviews-by-partner-id/{supplierId}/`
and `/widgets/partner-banner/{supplierId}/` — see
`components/konfetti-components.yml`.

## Pitfalls

- **Experiences resolve by permalink, not id.** `GET
  /v1/store/events/{bare-id}` returns 404. Use the `permalink` field.
- **Included relations nest under `data`** — `supplier.data`, not `supplier`.
- **Do not follow redirects.** A 302 to `api.gokonfetti.com/login` means you
  touched a protected endpoint; it is an auth failure dressed as a redirect.
- **`avg_review` is inconsistently typed** — a number on the event, a string on
  the category. Coerce before comparing.
- **500s happen** on `/add-ons` and `/calendar` even for valid experiences.
  Degrade gracefully rather than aborting the crawl, and check `content-type`
  before parsing an error body — 5xx returns HTML, not JSON.

See `data-model/konfetti-data-model.yml` for the full entity graph and
`conventions/konfetti-conventions.yml` for the pagination, include and filter
semantics.
