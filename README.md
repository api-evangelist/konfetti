# konfetti

konfetti (Konfetti GmbH, Berlin) operates a marketplace for bookable experiences — cooking classes, pottery and ceramics workshops, cocktail courses, tastings, boat tours, creative and craft workshops, DIY kits and team events — across Germany and Austria, with more than 7,600 listings. Alongside the consumer storefront it sells an all-in-one booking-management product to partner hosts, including an embeddable booking solution and marketing widgets.

Backed by: speedinvest

## A note on the API artifacts in this repository

konfetti publishes **no developer portal, no API reference and no OpenAPI document**. A JSON API does exist at `https://api.gokonfetti.com/v1`, and its `/v1/store/*` catalog endpoints answer without credentials, but it is an internal interface serving konfetti's own storefront.

Everything under `openapi/` and the artifacts derived from it was reconstructed by API Evangelist from direct probing of the live API and from konfetti's own publicly served frontend bundles, on 2026-07-19. Every status code recorded was observed. Nothing was invented.

There are no published terms of use covering programmatic access, no rate limits, no versioning policy beyond the `/v1` prefix, no deprecation policy, no status page and no developer support channel. Treat these artifacts as research material, not as a contract — and contact konfetti (hallo@gokonfetti.com) before building on them.

See `llms/konfetti-llms.txt` for a full index of what is in this repository.
