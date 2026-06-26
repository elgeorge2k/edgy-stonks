# Specification: Market Synchronization & Hydration

## 1. Runtime Context

This specification implements the `Lazy Evaluation & Async Hydration` principle from `CONSTITUTION.md`.

Market data synchronization is a hybrid client/server model:

- Angular controls request frequency through tab visibility state.
- The Cloudflare Worker enforces a 5-minute stale-first cache rule against Cloudflare D1.
- Third-party market-data fetches never block the synchronous user path.

## 2. Configurable Provider Endpoint

The Worker must read the market-data provider endpoint from configuration, for example `MARKET_DATA_PROVIDER_URL`.

Documentation must not hardcode provider-specific URLs. Massive.com provider data is intended for non-production PoC display compliance through the configured endpoint.

## 3. Frontend Sync Cadence

The Angular client controls how often it requests market data:

| Browser State | Sync Behavior |
| --- | --- |
| Visible and focused | Request market data at 1-minute granularity. |
| Hidden or unfocused | Request market data at 5-minute granularity. |
| Hidden for more than 15 minutes | Suspend polling until the tab becomes visible again. |
| Refocused | Resume immediately and trigger on-demand hydration if needed. |

## 4. Worker Cache and Hydration Flow

1. The client requests asset market prices from the Hono API.
2. The Worker reads `assets.last_updated_at` from D1.
3. If `current_time - last_updated_at <= 5 minutes`, the Worker returns cached asset data with `X-Cache-Status: Hit`.
4. If `current_time - last_updated_at > 5 minutes`, the Worker returns the stale cached asset data with `X-Cache-Status: Hydrating` and schedules D1 hydration asynchronously.
5. If the provider rate limit is reached or the provider is unavailable, the Worker returns the last cached price with `X-Cache-Status: Hydrating` and marks response metadata as delayed or cached rather than realtime.
6. If no cached asset data exists, the Worker returns a cache-miss response and schedules hydration asynchronously.

The hydration task bypasses the database task queue completely since market data is ephemeral and non-transactional. Instead, it executes as a lightweight, in-memory asynchronous background operation handled natively by the Cloudflare Workers runtime without blocking the synchronous HTTP response path:

```ts
// Triggered opportunistically in memory via the runtime context
ctx.waitUntil(
  fetch(MARKET_DATA_PROVIDER_URL)
    .then((response) => response.json())
    .then((data) => hydrateD1Assets(data))
);

## 5. Response Contract

| Header | Meaning |
| --- | --- |
| `X-Cache-Status: Hit` | Cached D1 data was fresh. |
| `X-Cache-Status: Hydrating` | Cached D1 data was stale and hydration is running asynchronously. |
| `X-Cache-Status: Miss` | No cached D1 data was available; hydration is scheduled asynchronously. |

## 6. Frontend UI Matrix

When the Angular frontend detects `X-Cache-Status: Hydrating`, it must temporarily lock volatile betting elements and present a randomized loading string combining the following structural arrays:

- **Actions:** Loading, Stowing, Booting, Reading, Lading, Arranging, Putting on, Taking on, Filling, Weighing-Down, Ballasting, Cramming, Burdening, Sophisticating, Doctoring, Debasing, Adulterating, Warping, Twisting, Perverting, Readying, Priming.
- **Targets:** cargo, the ball, freight, passengers, the stocks, an archipelago, the salesmen, the mop, the Everest, the Popocatepetl.
- **Suffix:** please wait..., hold your horses..., cool your jets..., keep your shirt on..., take a breather..., chill out..., sit tight..., easy does it...
