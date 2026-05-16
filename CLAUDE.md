# listmonk-stack
> Self-hosted Newsletter- und Marketing-Mail-Plattform für das FEVER-Ökosystem.

## Übersicht
- **Ziel:** Zentrales Versand-Backend für Newsletter, Post-Event-Follow-ups, Marketing-Funnels
- **Zielgruppe:** Toby (Admin) + alle Projekte als API-Konsumenten (feverevents-dj, später symbiosika, etc.)
- **Status:** [x] Neu / [ ] In Entwicklung / [ ] Wartung
- **Priorität:** Hoch
- **Deadline:** keine harte; FEVER EVENTS Post-Event-Follow-ups sind erster Use-Case

## Tech Stack
- **Service:** listmonk v4.1.0 (Go + Postgres, self-hosted, MIT-Lizenz)
- **DB:** PostgreSQL 17 (eigener Container, NICHT shared)
- **Hosting:** Coolify auf Hostinger VPS, Projekt "Infrastructure"
- **Domain:** `mailings.fever-context.de` (Wildcard *.fever-context.de via Traefik)
- **SMTP-Backend:** all-inkl/kasserver (info@fever-events.de)
- **Bilder:** Listmonk-eigene Media-Library (Volume `listmonk-uploads`)

## Projektstruktur
- `compose.yaml` — Listmonk + Postgres als Coolify-kompatibler Stack
- `.env.example` — Env-Slots (Coolify-Magic-Vars werden bevorzugt)
- `templates/` — wiederverwendbare HTML-Templates
  - `feverevents-post-event.tmpl` — FEVER EVENTS Post-Event-Follow-up
- `README.md` — Setup, Deploy, Admin-Erststart
- `CLAUDE.md` — diese Datei

## Wichtige externe Services
| Service | URL | Zweck |
|---|---|---|
| Coolify | `coolify.fever-context.de` | Deploy & Restart-Management |
| all-inkl kasserver | `w01XXXX.kasserver.com` | SMTP-Backend für `info@fever-events.de` |
| feverevents-dj | `dj.fever-events.de` | API-Consumer (Sync-Script: Gäste → Subscriber + Campaign-Draft) |
| Vault-MCP | Service-Gruppe `listmonk` | API-Tokens, Admin-Passwörter |

## Secrets (Vault-Gruppe `listmonk`)
| Key | Kategorie | Zweck |
|---|---|---|
| `coolify.app-uuid` | coolify | Coolify Application UUID |
| `coolify.db-uuid` | coolify | Coolify Postgres-Container UUID |
| `coolify.project-uuid` | coolify | Infrastructure-Projekt UUID |
| `coolify.domain` | coolify | mailings.fever-context.de |
| `admin_password` | credentials | Initial-Admin-PW (aus Coolify SERVICE_PASSWORD_LISTMONKADMIN) |
| `postgres_password` | credentials | DB-Passwort |
| `api_token_feverevents_dj` | credentials | API-Token für feverevents-dj Sync |
| `smtp_user` | credentials | all-inkl Mailbox-Login info@fever-events.de |
| `smtp_password` | credentials | all-inkl Mailbox-Passwort |

## Wichtige Entscheidungen
- **Eigener Postgres-Container** statt Shared-DB: Coolify-Pattern; Datenisolation; Postgres-17-alpine, Volume `listmonk-pgdata`
- **Subdomain unter fever-context.de** statt fever-events.de: Listmonk ist Infrastructure, Wildcard schon vorhanden, kein DNS-Setup
- **Versand über all-inkl statt eigener Mailserver**: schnell, vorhandenes Setup mit DKIM/SPF/DMARC; Brevo/Postmark als Fallback wenn Volumen wächst
- **Templates im Repo committed** (nicht nur in Listmonk-DB): Reproduzierbar, code-review-bar, in Backup
- **Build-time-Replace + Runtime-Variables**: Master-Template hat `{{EVENT_NAME}}` (Sync-Script ersetzt) + `{{ .Subscriber.Attribs.firstname }}` (Listmonk rendert pro Empfänger)

## Aktuelle TODOs
- [x] Compose-Stack + Config geschrieben
- [x] Polter-Template als Master-Template
- [ ] Coolify-Deploy via API auslösen
- [ ] Admin-Erststart im UI (Site-Name, Logo, From-Address)
- [ ] SMTP-Config (all-inkl, info@fever-events.de)
- [ ] API-User `feverevents-dj-bot` anlegen → Token in Vault
- [ ] Sync-Script in feverevents-dj-Repo (`scripts/sync-event-to-listmonk.mjs`)
- [ ] Polter-Liste & Campaign manuell befüllen für Erst-Versand

## Kurzbefehle (Dev)
```bash
# Lokal testen (optional)
docker compose up -d

# Logs
docker logs -f listmonk-app

# Backup-Dump
docker exec listmonk-db pg_dump -U listmonk listmonk > backup_$(date +%F).sql
```

## Backup
- CLAUDE.md + relevante Configs nach `infrastruktur/backup-claude-code/projects/listmonk-stack/` spiegeln (siehe globales CLAUDE.md).
- Postgres-Volume in Coolify Backup-Schedule aufnehmen sobald Daten relevant.
