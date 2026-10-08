# Ad revenue integration

The Android app now initializes Google Mobile Ads, requests consent before ad
requests, and reports production paid-event telemetry to the API. The API
stores idempotent estimated revenue events and exposes an admin summary.

## What is included

- Interstitial ad placement for eligible Bronze users, with a 15-minute cap.
- Google test inventory in debug builds; debug/test events are never reported
  as revenue.
- AdMob consent flow, with a privacy-options entry point.
- `POST /api/ads/revenue-events` for authenticated client telemetry.
- `GET /api/admin/ads/revenue-summary` for estimated telemetry totals.

The API summary is operational telemetry only. AdMob reports and payouts remain
the financial source of truth, because server-side verified revenue can differ
from a client paid event.

## Production configuration

1. Create and approve the Aidly Buddy app in the AdMob account and add its
   Android package name.
2. Create the production interstitial ad unit, verify the release ad-unit ID in
   `src/services/adService.ts`, and configure app-ads.txt on the verified
   developer website.
3. Configure the UMP/AdMob consent message for each region served by the app.
4. Store the AdMob application ID in the release CI secret `ADMOB_APP_ID`.
   Release Gradle builds intentionally fail when this secret is absent. Debug
   builds use Google's public test application ID.
5. Build a signed internal-test release, verify consent, ad loading, paid-event
   telemetry, and compare the API estimate with the AdMob console before a
   public rollout.

Never use test ad-unit IDs in a public build and never treat test impressions as
revenue.

