# Sentry Setup — What's Done & What Needs Improvement

**Branch**: `setup-sentry-tracking`
**PR**: #258
**Package**: `@sentry/nextjs ^8.55.0`

---

## What's Been Done (Current State)

### Files Added/Modified

| File | Purpose |
|------|---------|
| `sentry.client.config.ts` | Browser-side Sentry init — error tracking, performance tracing, and Session Replay |
| `sentry.server.config.ts` | Server-side (Node.js) init — captures SSR, API route, and server component errors |
| `sentry.edge.config.ts` | Edge runtime init — captures middleware and edge route errors |
| `instrumentation.ts` | Next.js instrumentation hook — loads the correct Sentry config per runtime, wires `onRequestError` |
| `app/global-error.tsx` | Root-level error boundary — reports unhandled errors to Sentry, shows fallback UI |
| `next.config.ts` | Wraps build with `withSentryConfig` for source map upload, hiding, and tree-shaking |
| `components/pendo-script.tsx` | Sets/clears `Sentry.setUser()` on login/logout alongside Pendo |
| `.env.example` | Documents `NEXT_PUBLIC_SENTRY_DSN` and `NEXT_PUBLIC_SENTRY_RELEASE` |
| `package.json` | Adds `@sentry/nextjs` dependency |

### Current Configuration Summary

**Client config:**
- `tracesSampleRate: 1` (100%)
- `replaysSessionSampleRate: 0.1` (10% of normal sessions replayed)
- `replaysOnErrorSampleRate: 1.0` (100% of error sessions replayed)
- Session Replay with `maskAllText: true`, `blockAllMedia: true`
- `networkDetailAllowUrls: [window.location.origin]`

**Server + Edge configs:**
- `tracesSampleRate: 1` (100%)
- `autoSessionTracking: true`

**Build config (`withSentryConfig`):**
- `org: 'dalgo'`, `project: 'javascript-nextjs'`
- `hideSourceMaps: true`, `widenClientFileUpload: true`, `disableLogger: true`
- `silent: !process.env.CI`

---

## What's Already Good

- **Replay sample rates** — `0.1` / `1.0` are Sentry's recommended defaults
- **`maskAllText` + `blockAllMedia`** — good privacy defaults for replay
- **`hideSourceMaps`**, **`widenClientFileUpload`**, **`disableLogger`** — all correct
- **`global-error.tsx`** — correctly wired as root error boundary
- **`instrumentation.ts`** — correctly uses `onRequestError` for server-side error capture
- **`Sentry.setUser`** — properly set on login and cleared on logout via `pendo-script.tsx`

---

## Improvements Needed

### HIGH Priority

#### 1. Lower `tracesSampleRate` from `1` to `0.2`
**Files**: `sentry.client.config.ts`, `sentry.server.config.ts`, `sentry.edge.config.ts`

100% tracing sends every page navigation, API call, and server request as a transaction. This burns through Sentry quota rapidly. A 20% sample is statistically representative enough to spot slow routes and performance regressions. Sentry's own docs recommend 10-20% for production.

```typescript
tracesSampleRate: 0.2,
```

#### 2. Add `ignoreErrors` to filter browser noise
**File**: `sentry.client.config.ts`

Without this, the Sentry issue feed gets dominated by non-actionable noise:

```typescript
ignoreErrors: [
  // Benign browser behavior — fires when a resize callback takes too long
  /ResizeObserver loop/,
  'ResizeObserver loop completed with undelivered notifications',

  // Third-party scripts rejecting promises with non-Error objects
  'Non-Error promise rejection captured',
  'Non-Error exception captured',

  // Transient network issues on user devices (flaky WiFi, going offline)
  'Failed to fetch',
  'Load failed',
  'NetworkError',
  'Network request failed',
  /^TypeError: cancelled$/,

  // User navigated away before a fetch completed
  'AbortError',

  // Users on stale tabs after a deploy try to load chunks that no longer exist
  /Loading chunk [\d]+ failed/,
  'ChunkLoadError',
],
```

#### 3. Fix `window.location.origin` SSR risk
**File**: `sentry.client.config.ts`

`window.location.origin` is accessed at module level. During the Next.js build the bundler may evaluate this in a Node.js context where `window` doesn't exist, causing a crash.

**Option A** — guard with `typeof window`:
```typescript
networkDetailAllowUrls: [
  typeof window !== 'undefined' ? window.location.origin : '',
].filter(Boolean),
```

**Option B** — use a regex pattern (preferred, avoids the issue entirely):
```typescript
networkDetailAllowUrls: [
  /^https?:\/\/(.*\.)?dalgo\.in/,
],
```

#### 4. Add `SENTRY_AUTH_TOKEN` for source map uploads
**Where**: CI/CD environment (GitHub Actions secrets)

Without an auth token, the `withSentryConfig` build plugin cannot upload source maps to Sentry. Stack traces in production will show minified code like `main-a3f8b2.js:1:28492 n()` instead of `ChartBuilder.tsx:142 handleSave()`.

Steps:
1. Generate token in Sentry: **Settings → Auth Tokens**
2. Add `SENTRY_AUTH_TOKEN` as a GitHub Actions secret
3. Add `SENTRY_AUTH_TOKEN=` to `.env.example` for documentation
4. The plugin reads `SENTRY_AUTH_TOKEN` from the environment automatically — no code change needed

---

### MEDIUM Priority

#### 5. Add `enabled` flag to disable Sentry in development
**Files**: `sentry.client.config.ts`, `sentry.server.config.ts`, `sentry.edge.config.ts`

Without this, Sentry initializes locally, adding dev server overhead and potentially polluting the production Sentry project if someone sets a DSN locally.

```typescript
Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  enabled: !!process.env.NEXT_PUBLIC_SENTRY_DSN,
  // ... rest of config
});
```

No DSN → no Sentry. Clean toggle.

#### 6. Add `denyUrls` to block browser extension errors
**File**: `sentry.client.config.ts`

Browser extensions (password managers, Grammarly, ad blockers, translation tools) inject scripts into your page. When their code throws, the browser attributes it to your domain.

```typescript
denyUrls: [
  /extensions\//i,
  /^chrome:\/\//i,
  /^chrome-extension:\/\//i,
  /^moz-extension:\/\//i,
  /^safari-extension:\/\//i,
  /^safari-web-extension:\/\//i,
],
```

#### 7. Add `tunnelRoute` to bypass ad blockers
**File**: `next.config.ts`

Ad blockers block requests to `sentry.io`. `tunnelRoute` proxies Sentry events through your own server (`yourdomain.com/monitoring` → Sentry), so ad blockers don't interfere. One-line change, no downside.

```typescript
export default withSentryConfig(nextConfig, {
  // ... existing config
  tunnelRoute: '/monitoring',
});
```

Without this, you're blind to errors from ~10-30% of users who use ad blockers.

#### 8. Add `beforeSend` for programmatic error filtering
**File**: `sentry.client.config.ts`

`ignoreErrors` matches on error message strings. `beforeSend` gives programmatic control for cases that need more nuance:

```typescript
beforeSend(event, hint) {
  const error = hint.originalException;

  // Drop AbortError — user navigated away, request cancelled
  if (error instanceof Error && error.name === 'AbortError') {
    return null;
  }

  return event;
},
```

---

### LOW Priority

#### 9. Remove `autoSessionTracking` from server/edge configs
**Files**: `sentry.server.config.ts`, `sentry.edge.config.ts`

Sessions are a browser concept (tab open → tab close). On the server, `autoSessionTracking` is deprecated and being removed in SDK v9. It's a no-op on current versions. Removing it keeps the config clean and avoids a future breaking change.

---

## Future Considerations (Not Urgent)

- **`tracesSampler` function** — graduate from flat `tracesSampleRate` to a `tracesSampler` function for per-route sampling (e.g., health checks at 1%, API routes at 50%, default 20%)
- **`thirdPartyErrorFilterIntegration`** — available since SDK v8.10.0, automatically marks your own code during build and drops errors from third-party frames. More robust than manual `denyUrls` but requires adding `applicationKey` to `withSentryConfig`
- **`allowUrls`** — restrict error capture to only your own domain. More aggressive than `denyUrls`. Defer unless `denyUrls` isn't catching enough noise
- **Route-specific `error.tsx`** — add `error.tsx` boundaries in key route segments (e.g., `app/charts/error.tsx`) for more granular error handling with better UX than the bare-bones `global-error.tsx`
