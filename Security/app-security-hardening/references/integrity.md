# Device integrity: Play Integrity (Android) + App Attest (iOS)

Proof, signed by Google or Apple, that the request comes from **your genuine app** on a
**non-tampered device**. The server issues a short pass (1 h) that sensitive requests (payments,
buying, selling) must carry when enforcing.

## Flow

1. `GET /integrity` → `{mode: off|monitor|enforce, android: bool, ios: bool}`.
2. `POST /integrity/challenge` → HMAC-signed challenge bound to the user and a timestamp.
3. Android: native `IntegrityManager.requestIntegrityToken(nonce = base64url(challenge))`
   (classic) → `POST /integrity/android {challenge, token}`.
   Server decodes with Google: `POST https://playintegrity.googleapis.com/v1/<package>:decodeIntegrityToken`
   using a service account access token with scope `https://www.googleapis.com/auth/playintegrity`
   (the Firebase service account works).
   Accept only if: nonce matches, `requestPackageName` and `appIntegrity.packageName` = yours,
   `appRecognitionVerdict == PLAY_RECOGNIZED`, device verdict contains `MEETS_DEVICE_INTEGRITY`
   (or `MEETS_STRONG_INTEGRITY`).
4. iOS: `DCAppAttestService` — generate key once per install, `attestKey(clientDataHash =
   sha256(challenge))` → `POST /integrity/ios/attest`; later passes use `generateAssertion` →
   `/integrity/ios/assert`. Server verifies the CBOR attestation against **Apple's App Attest root
   CA** (bundle the PEM; make sure `.gitignore` doesn't drop `*.pem` for it), checks App ID
   `TEAMID.bundle.id`, stores the public key and counter, rejects counter replays and keys used by
   another user.
5. Server returns `{pass, expiresAt}` (HMAC, bound to user + platform). The app sends it in a header
   on every request; `requireIntegrity` on sensitive routes checks it.

## Modes

- `monitor` (start here): requests without a pass are logged, never refused.
  **The app must still run the check** (once per launch, in the background, a few seconds after
  start) — otherwise monitor mode collects nothing. Save every *failed* verdict as a security event
  (`integrity_device_not_trusted`, `integrity_unrecognized_app`, `integrity_wrong_app`,
  `integrity_invalid_proof`) and show them on the admin Security page.
- `enforce`: sensitive routes return `403 integrity_required`; the app gets a pass and retries once.
  Switch only after a week or more of monitor data shows real users pass. Admins bypass.

## Console setup (the user does these; give exact clicks)

- **Google Cloud** (the project of the service account): enable **Google Play Integrity API**.
- **Play Console → Protected with Play → Play Integrity API**: link that Cloud project. In
  "Change responses" the defaults are enough (App integrity + Device integrity on). Extra verdicts
  (recent activity, device attributes, Play Protect, app access risk) only help if the server reads
  them.
- **Apple:** App Attest capability on the App ID; entitlement
  `com.apple.developer.devicecheck.appattest-environment` = `production` for release.
- Server env: package name, `APP_ATTEST_APP_ID=TEAMID.bundle.id`, mode.

## Verify it works

- From the server, call decodeIntegrityToken with a fake token: **400 INVALID_ARGUMENT** means the
  API is enabled and the service account is authorized; 403 means not enabled / not linked.
- Unit-test the verdict judge and the routes with a fake decoder and a fake Apple CA (build a test
  root + leaf with node-forge or @peculiar/x509).
- After the app release: watch the Security page and the "integrity passed" log lines.

## Gotchas

- Sideloaded/debug builds are `UNRECOGNIZED_VERSION` — expected; test enforce with a Play
  internal-testing build.
- Lost upload key: Play App Signing means users are unaffected; request an upload key reset
  (Protected with Play → Play Store protection → Manage Play app signing) with the new key's
  `upload_certificate.pem`.
- Quotas: 10k Play Integrity calls/day by default; once per launch + 1 h passes stays well under.
