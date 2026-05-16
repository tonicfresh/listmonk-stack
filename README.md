# listmonk-stack
> Self-hosted Newsletter- und Marketing-Mail-Plattform für das FEVER-Ökosystem.

## Übersicht
- **Service:** [listmonk](https://listmonk.app/) (Go + Postgres, self-hosted)
- **Domain:** `mailings.fever-context.de`
- **Coolify-Projekt:** Infrastructure (`hk488c8k4k0sk8s40s0044co`)
- **Zweck:** Newsletter, Post-Event-Follow-ups, künftig Marketing-Funnels — zentrales Versand-Backend für alle Projekte (FEVER EVENTS, Symbiosika optional)
- **Trennt klar von:** `mail-app` (= IMAP-Inbox-Client) und `feverevents-dj`-internem Mailer (= transactional: OTP, Recovery)

## Architektur
```
   ┌─────────────────────────┐
   │  feverevents-dj         │
   │  (Next.js)              │
   │                         │
   │  guests (Postgres)      │──┐
   │  played_tracks          │  │  Sync-Script:
   │  events                 │  │  Push Subscriber + Campaign-Draft
   └─────────────────────────┘  │  via Listmonk-REST-API
                                ▼
   ┌──────────────────────────────────────────┐
   │  listmonk-stack (Coolify)                │
   │  mailings.fever-context.de               │
   │                                          │
   │  listmonk (Go) ── Postgres (eigener)     │
   │      │                                   │
   │      └── SMTP: all-inkl kasserver        │
   │          From: info@fever-events.de      │
   └──────────────────────────────────────────┘
                                │
                                ▼
                          Empfänger
```

## Komponenten
| Service | Image | Port | Beschreibung |
|---|---|---|---|
| `listmonk` | `listmonk/listmonk:v4.1.0` | 9000 | Web-UI + REST-API |
| `listmonk-db` | `postgres:17-alpine` | 5432 (intern) | Eigene DB, NICHT shared |

## Deploy via Coolify (One-Off)
1. Coolify → Infrastructure → **+ Add Resource** → **Docker Compose**
2. Source: **Public Repository** → `https://github.com/tonicfresh/listmonk-stack` (Branch `main`)
3. Compose-File-Pfad: `compose.yaml`
4. Domain: Coolify generiert via Wildcard `mailings.fever-context.de` (Service `listmonk`, Port 9000)
5. Environment-Vars: keine manuellen — Coolify-Magic-Vars reichen
6. **Deploy** klicken — listmonk läuft inkl. Postgres-Migrations beim ersten Start

## Erst-Setup (im Listmonk-UI)
1. `https://mailings.fever-context.de` öffnen
2. Login: `admin` / `<SERVICE_PASSWORD_LISTMONKADMIN aus Coolify-Env>`
3. **Settings → General**
   - Site name: `FEVER EVENTS Mailings`
   - Logo URL: `https://www.fever-events.de/images/logo.svg`
   - Site URL: `https://mailings.fever-context.de`
4. **Settings → SMTP**
   - Host: all-inkl kasserver (`w01XXXX.kasserver.com`)
   - Port: `465` (SSL/TLS)
   - Auth: LOGIN
   - User: Mailbox-Login `info@fever-events.de`
   - Password: Mailbox-Passwort
   - From: `FEVER EVENTS <info@fever-events.de>`
   - Max concurrent connections: `5` (all-inkl Limit beachten)
   - Send timeout: `15s`, Retries: `3`
5. **Settings → Performance** → Concurrency: `2`, Batch size: `100`
6. **Users** → API-User anlegen: `feverevents-dj-bot` mit Permissions `lists:manage`, `subscribers:manage`, `campaigns:manage`. Token ins Vault speichern: `sm vault store listmonk.api_token_feverevents_dj <token>`.

## Templates
- `templates/feverevents-post-event.tmpl` — Post-Event-Follow-up für FEVER EVENTS (FEVER-Brand: `#100726` BG, Pink `#D71362`, Türkis `#00CFC8`)

## Sync-Script (in feverevents-dj-Repo)
Pfad: `feverevents-dj/scripts/sync-event-to-listmonk.mjs`

Bei jedem neuen Event:
```bash
node scripts/sync-event-to-listmonk.mjs --event=polter-jens-svenja-2026-05-15
```
- Legt Liste `event-<slug>` an (idempotent)
- Pusht alle `guests` mit `consent_followup=true` + E-Mail als Subscriber
- Erstellt eine Campaign-Draft mit Template `feverevents-post-event`, Subject, Spotify-Link, Google-Review-Link aus Event-Daten
- Toby öffnet sie im Listmonk-UI, prüft, sendet

## Backup
- DB: `listmonk-pgdata` Volume → in Coolify Backup-Schedule aufnehmen
- Uploads: `listmonk-uploads` Volume → bei Bedarf nach S3/MinIO syncen

## Kosten-Tracking
- Listmonk selbst kostet nichts (self-hosted)
- SMTP über all-inkl: in deinem bestehenden Hosting-Paket enthalten
- Bei >2000 Mails/Tag: Brevo/Postmark als SMTP-Backend einbinden

## CLAUDE.md-Spiegelung
Backup nach `infrastruktur/backup-claude-code/projects/listmonk-stack/CLAUDE.md`.
