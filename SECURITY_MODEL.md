# Threat model: melissadu818/juice-shop

Sources: repository `threat-model.json` (OWASP Threat Dragon "High Level Data Flow"), `SECURITY.md`, `AGENTS.md`, and the code at the scanned commit. No infrastructure repository; deployment facts come from `Dockerfile` and `config/*.yml`.

## System overview

OWASP Juice Shop is an **intentionally vulnerable** web shop used for security training, CTFs and tool demos. A single Node.js/Express process (`server.ts`, `app.ts`) serves the Angular SPA, a REST/Finale API over Sequelize/SQLite and MarsDB, static file trees, and a WebSocket channel for challenge notifications. It ships as a distroless container (`Dockerfile`, runs as UID 65532, port 3000).

Per `SECURITY.md` (project-authored policy): vulnerabilities that are part of hacking challenges are intended; only vulnerabilities **not** part of any challenge are treated as real security issues. Intended vulnerabilities are marked in code with `// vuln-code-snippet start|vuln-line <challengeKey>` and catalogued in `data/static/challenges.yml` (112 challenges); solved via `challengeUtils.solveIf`.

## Trust boundaries (from threat-model.json)

1. Browser (B2C customer, B2B customer, Admin, Accounting) -> Application Server (internet boundary).
2. Application Server -> data stores (SQLite, MarsDB, local file system).
3. Application Server -> external services: Google OAuth (out of scope in threat-model.json), optional LLM endpoint (`application.chatBot.llmApiUrl`, default `http://localhost:11434/v1`), Web3 RPC, outgoing webhook (`lib/webhook.ts`).
4. Container / host boundary (`Dockerfile`, safety mode in `lib/utils.ts` disables dangerous challenges on Docker/Heroku unless `safetyMode: disabled`, e.g. `config/unsafe.yml`).

## Components

| Name | Code | Notes |
|---|---|---|
| Web Client | `frontend/` | Angular SPA, Hacking Instructor, score board |
| API | `server.ts`, `app.ts`, `routes/` (shop, basket, order, reviews, B2B `/b2b/v2`, profile), `models/`, `views/` | Finale REST resources at `/api/*`, custom routes at `/rest/*` |
| Authentication | `lib/insecurity.ts`, `routes/login.ts`, `routes/2fa.ts`, `routes/resetPassword.ts`, `routes/changePassword.ts`, `routes/currentUser.ts` | RS256 JWT, key in `encryptionkeys/`; role checks `isAuthorized`, `denyAll`, `isAccounting`, `appendUserId` |
| File Serving | `routes/fileServer.ts`, `routes/fileUpload.ts`, `routes/profileImage*.ts`, `routes/logfileServer.ts`, `routes/keyServer.ts`, `routes/quarantineServer.ts`, `ftp/`, `uploads/` | `/ftp`, `/encryptionkeys`, `/support/logs`, uploads (XXE/YAML/zip) |
| Challenge Engine | `lib/challengeUtils.ts`, `lib/antiCheat.ts`, `lib/codingChallenges.ts`, `lib/startup/`, `routes/verify.ts`, `routes/vulnCode*.ts`, `routes/continueCode.ts`, `routes/restoreProgress.ts`, `data/`, `rsn/` | Challenge solving, scoring, coding challenges, startup checks, seed data |
| Chatbot & Web3 | `routes/chat.ts`, `routes/checkKeys.ts`, `routes/web3Wallet.ts`, `routes/nftMint.ts`, `frontend/src/app/{chatbot,wallet-web3,web3-sandbox,faucet,nft-unlock}` | LLM chatbot (prompt injection challenges), Bee Haven / Web3 challenges |
| Build & CI | `Dockerfile`, `Gruntfile.js`, `package.json`, `.github/workflows/`, `config/` | Image build, SBOM, release pipelines |

## Entry points

- Unauthenticated HTTP: SPA, `/rest/user/login`, `/rest/products/search`, `/api/Users` (register), `/api/Feedbacks`, `/rest/track-order/:id`, `/rest/chat`, `/ftp`, `/encryptionkeys`, `/support/logs`, `/api-docs`, `/file-upload`, `/redirect`, `/metrics`, `/rest/web3/*`.
- Authenticated (JWT): basket, orders, addresses, cards, wallet, reviews, data export, profile image upload by file/URL, B2B orders, 2FA.
- Role-gated: accounting (`/rest/order-history/orders`, `/api/Quantitys/:id`), admin UI (client-side route only).
- Build/CI: GitHub Actions workflows in `.github/workflows/` (run on push/PR to the fork).

## Assets

- Challenge integrity: solutions, `ctf.key` / continue codes, coding-challenge fixes (CTF scoring fairness).
- Seeded user accounts and credentials (fictional), JWT private key in `lib/insecurity.ts` / `encryptionkeys/` (intentionally exposed).
- The host / container running the instance (RCE, sandbox escape beyond what challenges intend, resource exhaustion).
- CI secrets and release artifacts (`.github/workflows/`), the published container image and SBOM.

## Attacker profiles

- Training participant / CTF player: unauthenticated or registered user hacking the instance; expected and invited for challenge vulnerabilities.
- Opportunistic internet attacker against an exposed instance: not applicable to this fork per the user (local/training only), kept for completeness.
- Supply-chain / CI attacker: malicious PR or dependency targeting workflows and the released image.

## Existing controls

- Safety mode (`challenges.safetyMode: auto`) disables dangerous challenges on Docker/Heroku.
- Distroless image, non-root UID, `helmet.noSniff`/`frameguard`, rate limit on reset-password, `denyAll` on sensitive Finale resources.
- Intended weaknesses are deliberately unmitigated (e.g. hard-coded HMAC key, MD5 password hashes, `cookieParser('kekse')`, open CORS).

## User decisions (2026-11-13)

- **Every finding is treated as real**, including intended challenge vulnerabilities: this fork is handled as a real app, so `SECURITY.md`'s "challenge vulnerabilities are intended" policy is **not** applied as accepted risk here. Challenge markers are informational only.
- **Deployment: local / training use only.** No internet-exposed instance; the reachable attackers are users of the local/training instance.
- **No infrastructure repository**; deployment facts are limited to `Dockerfile` and `config/`.
- Finding tags wanted: OWASP Top 10 category, CWE (plus Component).

## Out of scope / accepted risk

- Google OAuth flow (`threat-model.json`, project-authored, marks Google and the OAuth2 flow out of scope).
- Nothing else is ruled out by the user.

## Open questions

- Which assets matter most beyond the defaults above (e.g. CI secrets vs. instance integrity)?
