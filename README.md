# vumlo-ucu

The Українського католицького університету deployment of [Vumlo](https://github.com/Vumlo/vumlo-core).
This repository holds only what is UCU's own; the product is a released, checksummed
source archive of vumlo-core, fetched here and never edited.

## Layout

| Path | What |
|---|---|
| `core.version`, `core.sha256` | the vumlo-core release in use and the SHA-256 of its archive |
| `vumlo.toml` | the configuration: name, short name `UCU`, language `uk`, time zone `Europe/Kyiv`, sign-up only for `ucu.edu.ua`, no optional modules |
| `brand/logo.png` | the UCU monogram, from the university's logo set (the `institution.logo` of `vumlo.toml`) |
| `compose.ucu.yaml` | UCU's additions on top of core's own `deploy/compose.yaml` (a label with the core version on every image) |
| `.env.example` | the secrets, as a template. `.env` is never committed |
| `vendor/vumlo-core/` | the vumlo-core release of `core.version`, unpacked as it is (`backend/` the API and worker, `web/` the web app, `deploy/`). Committed so that Vercel can build `web/`; never edited, `scripts/fetch-core` replaces it |
| `.vumlo/generated/` | the branding (`client-config.json`, icons, manifest) made from `vumlo.toml` and `brand/`; committed because Vercel has no `vumlo-cli`. Refresh it with `scripts/brand` |
| `.github/workflows/desktop-release.yml` | builds the UCU desktop app (Linux, Windows, macOS) from `vendor/vumlo-core` and `vumlo.toml`; copied from core's `desktop/release/`, see below |
| `aws/` | `stack.yaml` (CloudFormation: the EC2 instance for the API, worker and PostgreSQL, ECR, IAM) and `Caddyfile` (HTTPS for the API domain) |
| `scripts/fetch-core` | downloads the pinned release, checks it, unpacks it into `vendor/vumlo-core` |
| `scripts/brand` | `brand apply`, then copies the result to `.vumlo/generated` |
| `scripts/bump-core <version>` | moves to another core release; undoes itself if anything fails |
| `scripts/vumlo` | runs `vumlo-cli` or `docker compose` for UCU (`scripts/vumlo doctor`, `scripts/vumlo compose ps`) |
| `scripts/smoke-test` | builds and runs a throwaway installation and checks it |

## Build and run

```sh
cp .env.example .env && chmod 600 .env     # set one password in POSTGRES_PASSWORD and DATABASE_URL
scripts/fetch-core                         # also done by scripts/vumlo when vendor/ is missing
scripts/vumlo config check
scripts/vumlo brand apply
scripts/vumlo compose build                # compiles the Rust core: needs several GB of RAM or swap
scripts/vumlo compose up -d --wait postgres
scripts/vumlo db migrate
scripts/vumlo admin create you@ucu.edu.ua --name Name --surname Surname   # asks for the password
scripts/vumlo compose up -d
scripts/vumlo doctor
```

`scripts/fetch-core` fails, and changes nothing, if the archive's SHA-256 is not the one in
`core.sha256`, is not in the `SHA256SUMS` of that GitHub Release, or the unpacked version is not
`core.version`. The full guide to what the commands do is `vendor/vumlo-core/docs/install.md`.

**Before this goes anywhere real** change two values in `vumlo.toml`: `server.public_origin` is
`http://localhost:3000` and `[auth] cookie_secure = false` is for a trial over plain HTTP. Set the
real `https://` address (behind Caddy, `vendor/vumlo-core/deploy/Caddyfile`) and delete the
`cookie_secure` line. `institution.mode` is `university` and cannot be changed after the first
start. No `[mail]` is configured, so sign-up by email and password reset are off until it is.

## Vercel (the web app)

Import this repository as a Vercel project and set:

- **Root Directory:** `vendor/vumlo-core/web` (with "Include source files outside of the Root
  Directory" on). Framework: Next.js. Install and build commands: the defaults (`npm ci`, `npm run build`).
- **Environment variables:**
  - `VUMLO_GENERATED_DIR` = `../../../.vumlo/generated` (where the build reads the branding)
  - `VUMLO_API_ORIGIN` = `https://<the API domain>` (where `/api/v1` is proxied: the AWS back end)
- **Domain:** the address in `server.public_origin` of `vumlo.toml`. The branding was generated for
  that address; if you change it, run `scripts/brand` and commit `.vumlo/generated`.

Only the web app runs on Vercel. The API, the worker and PostgreSQL run on AWS (`aws/`).

## Desktop app

The same web app in a Tauri window, with offline reading and drafts (`vendor/vumlo-core/docs/desktop.md`). Its name,
icon and server come from `vumlo.toml` and `brand/`; its links are `vumlo-ucu://...`. A tag `desktop-v1.0.0` builds
the installers in GitHub Actions and puts them in a **draft** Release; without signing secrets (`WINDOWS_CERTIFICATE`,
`APPLE_CERTIFICATE`, ...) they are unsigned, and the docs say what people see. Publish the draft, then in the web app
(Admin, Institution, Desktop app) paste the address of each installer: the download notice and page appear.

## Release procedure

A release is a tag of this repository, `YYYY.M.N`, and the server runs **from the tag only**, never
from a branch.

1. **Bump** (when core has a new release): `scripts/bump-core 0.1.1`. It writes `core.version` and
   `core.sha256`, fetches the archive, runs `config check` and `brand apply`, builds, and commits.
   If a step fails it restores everything and commits nothing. Read the core `CHANGELOG.md` **Upgrade
   notes** of the new release first and change `vumlo.toml` if they say so.
2. **Config check**: `scripts/vumlo config check` (the bump ran it; run it again after any edit).
3. **Build and smoke test**: `scripts/smoke-test` builds from this tree and checks `doctor`,
   `GET /institution`, the web app and a login, in a throwaway installation.
4. **Tag**: `git tag -a 2026.10.2 -m "UCU 2026.10.2, vumlo-core 0.1.1" && git push origin main 2026.10.2`.
5. **Deploy from the tag only**: on the server `git clone --branch 2026.10.2 <this repository>` (or
   `git fetch --tags && git checkout 2026.10.2`), put `.env` in, then
   `scripts/fetch-core && scripts/vumlo config check` (this also copies `vumlo.toml`, `brand/` and
   `.env` into `vendor/vumlo-core`), and `cd vendor/vumlo-core && deploy/update.sh --docker`: core's
   own script checks the configuration, applies the brand, builds, backs up and reads the dump
   back, stops, migrates, starts, and runs `doctor`. A first installation uses the "Build and run"
   commands above instead.

Needs on the machine: Docker with Compose 2.24 or newer, `curl`, `openssl`, `sha256sum`, `git`.
