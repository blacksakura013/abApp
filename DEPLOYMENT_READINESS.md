# Aidly Buddy deployment readiness

## Verified in this workspace

- API unit tests, `go vet`, and `govulncheck` pass with Go 1.25.13.
- Web TypeScript and Next.js production build pass.
- Mobile TypeScript and ESLint pass; Android debug build has been proven from a short path (see `abAppMobile/RELEASE.md`).
- PostgreSQL integration tests cover KYC approval, verified worker discovery, idempotent booking, settlement and balanced minor-unit ledger, GPS check-in/out, incident creation, and chat; test data runs inside a rolled-back transaction.
- Production configuration fails closed when secrets or providers are missing.

## Required before a real-user production launch

These values must be supplied through the deployment secret manager; never commit them:

- PostgreSQL TLS connection and credentials (`DB_*`, `DB_SSL_MODE=require` or stronger).
- Redis password and endpoint (`REDIS_*`).
- Private S3/R2-compatible KYC storage credentials (`STORAGE_*`).
- SMS and email delivery webhook URLs/secrets.
- Google and Facebook production app credentials.
- Explicit production `ALLOWED_ORIGINS`.
- Initial admin email/password and Base32 TOTP secret (password minimum 12 characters; enroll the same TOTP secret in the operations authenticator and rotate bootstrap credentials).
- Payment provider create-intent URL, API secret, and webhook signing secret.
- Android upload keystore variables listed in `abAppMobile/RELEASE.md`.
- Apple signing/profile and a macOS build host for iOS.

## Mandatory launch gates

- Select and contract a Thailand-compatible payment provider and complete sandbox plus settlement/reconciliation UAT. The API implements a generic signed provider contract, but cannot move real funds without a provider account.
- Select a KYC/liveness provider or formally approve trained manual review. Current capture stores private ID/passport and three face-angle evidences; it does not claim certified biometric liveness or document authenticity.
- Obtain PDPA/legal approval for privacy notice, consent versions, sensitive health/biometric processing, retention, data-subject requests, and the 30-day account deletion workflow.
- Configure Firebase APNs/FCM credentials and exercise push delivery on physical Android and iOS devices.
- Run end-to-end pilot acceptance on production-like infrastructure: registration, OTP, KYC review, worker activation, booking, payment webhook, chat, check-in/out, refund, and audit export.

## Local deployment validation

From `abappapi`:

```powershell
docker compose --env-file .env.production config --quiet
docker compose --env-file .env.production up --build -d
```

The first command is expected to fail if a required secret is absent. Do not replace missing values with sample secrets.
