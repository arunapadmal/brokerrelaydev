# HTTPS QR invitation fix — development integration notes

Status: **Integration blocked pending application source, Android signing details, and deployment ownership verification.** No production deployment authorised.

## Repository audit (23 November 2026)

The `arunapadmal/brokerrelaydev` default branch contains `_next/static/` compiled Next.js assets, images, a CSV example, and `netlify.toml`. It does **not** contain `package.json`, `app/`, `src/`, `public/`, Flutter `android/` or `ios/`, or Supabase Functions source. The existing `netlify.toml` uses `npm run build` and publishes `.next`, so adding isolated static files here would not establish a working Next.js route or serve association files. Do not change routing in this repo without restoring the authoritative source and Netlify configuration.

## Required behaviour

- Canonical new invitation URL: `https://brokerrelay.com.au/join?token=<opaque-random-token>`.
- Render an accessible landing page over HTTPS when the operating system does not open the app. Never auto-redeem a token on GET, preview, or link unfurl.
- The page may display generic instructions initially. Any broker/company detail requires server-side token validation and safe disclosure rules.
- Open the client app using verified HTTPS links (Android App Links and iOS Universal Links). Provide Play Store / App Store links only when the canonical app listing is confirmed; no forced store redirect.
- Existing `aidezconnect://join?token=...` links remain supported temporarily in the mobile handler.
- Invitation exchange must validate expiry, status, target organisation/broker, authentication, and optional intended recipient; redeem exactly once in a transaction, never from frontend-only checks.
- Hash tokens in Supabase; do not log raw tokens, print them in analytics, or put them in error telemetry; short TTL and replay protection. Referrer-Policy: no-referrer, Cache-Control: no-store on the join page and token-bearing responses.
- Auth/login/signup transitions must preserve invite context without exposing tokens to unrelated origins.
- QR generators should encode the HTTPS URL using the same invitation service; changing the QR code styling alone cannot resolve deep linking.

## Android integration (Flutter repository)

1. Locate app `applicationId` and actual launch Activity. Add an exported `VIEW` intent-filter with `DEFAULT`, `BROWSABLE`, `android:autoVerify="true"`, `scheme="https"`, `host="brokerrelay.com.au"`, `pathPrefix="/join"`. Keep existing custom scheme for compatibility.
2. Publish `https://brokerrelay.com.au/.well-known/assetlinks.json` with:
   `[{"relation":["delegate_permission/common.handle_all_urls"],"target":{"namespace":"android_app","package_name":"<verified-client-app-package>","sha256_cert_fingerprints":["<verified-Play-App-Signing-SHA256>"]}}]`
   **Do not deploy placeholders.** Use the actual *client* app package, not the broker Pro package. Include all required certs only if verified.
3. In Flutter `app_links`, handle initial and foreground links, accept only expected HTTPS host/path (plus legacy scheme), guard duplicate events, pass the token to the invitation flow, and preserve it through auth. Handle invalid/expired links gracefully.

## iOS integration

- Add Associated Domains entitlement `applinks:brokerrelay.com.au` to the correct client-app target and provisioning.
- Serve `/.well-known/apple-app-site-association` as JSON without a redirect, containing the verified Apple Team ID + bundle ID and `/join` match only.
- Test app-open, browser fallback, and Safari same-domain navigation behaviour.

## Next.js and Netlify integration

- Implement `app/join/page.tsx` and token-safe invitation handoff in the authoritative web source (not this compiled-assets repo).
- Serve Android/iOS association files from the deployment that *actually owns* `brokerrelay.com.au`. `assetlinks.json` on a test subdomain does **not** verify production host links; testing requires a matching configured test host or direct development device verification.
- Confirm no catch-all redirect intercepts `/join` or `/.well-known/*`. Check headers and response content type.

## Supabase integration

- Inspect current invite schema and Edge Function before changing code; no assumption about column names or redeem RPC.
- Update only **newly generated** invitations to use an environment-specific HTTPS base URL, configured server-side; preserve existing valid tokens, revocation and audit.
- Add tests for correct tenant/broker association, expired/revoked/used tokens, concurrent redemption, and unauthorised cross-tenant access.
- Avoid overwriting live Supabase secrets or deploying functions to production.

## Test gate

- Android internal-testing build with correct Play signing certificate; Android camera and Chrome; installed and not installed.
- iPhone via appropriate development/TestFlight build; camera, Safari and app-open behaviour.
- QR code scans, first-time account registration, existing login, first-password change, expired/invalid/used tokens, repeat scans and exact company/broker linkage.
- CI build/lint and integration tests pass in development before any merge or Netlify/Store/Supabase production deployment.

## Blockers

1. Authoritative Next.js website source and which Netlify site/domain deployment serves brokerrelay.com.au.
2. Flutter client app source, including Android/iOS native configuration.
3. Supabase invitation generation and redemption source or development project inspection.
4. Client package name, Google Play app-signing SHA-256, Apple Team ID and bundle ID.

**No production changes, DNS changes, app publishing, or secret modifications authorised in this branch.**
