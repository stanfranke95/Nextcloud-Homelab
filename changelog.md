# Changelog

A dated log of real changes made to this homelab. Newest entries at the top.
Kept as a running record rather than rewritten after the fact — if something
was tried and reverted, or done in a messier order than looks clean in
hindsight, it stays that way here.

---

## 2026-09-25
- Repo created and portfolio documentation started: `README.md`,
  `architecture.md`, `incident-report.md`, `known-issues.md`.
- Ran a health check after ~3 weeks with no hands-on time — confirmed both
  nodes healthy, Uptime Kuma green, backups intact.
- Noted Nextcloud 32.0.14 has reached end-of-life (Sept 2026); upgrade to
  33.x logged as a known issue rather than done immediately.

## 2026-08-28
- **Major triage session** after discovering Pi services down following a
  power outage 4–5 days prior. Full writeup in
  [`incident-report.md`](./incident-report.md).
- Fixed Portainer on the Pi: recreated with a proper `--restart
  unless-stopped` policy and correct port mappings (it had neither).
- Diagnosed and fixed a boot-order race condition causing Prometheus and
  Grafana to lose their port mappings after a power event.
- Found and fixed a backup script that had been silently writing to the
  wrong destination path for weeks, filling the Pi's OS drive to 100%.
  Corrected the path, fixed `backupuser` directory ownership, ran and
  verified a full manual backup, and cleaned up 8.7GB of failed backup data.
- Cleaned up two dormant test-restore stacks and their orphaned volumes on
  the main server (leftover from earlier restore-verification testing).
- Fixed Nextcloud's missing database indices and pending mimetype migration
  via `occ` commands.
- Fixed a missing cache directory causing recurring `JSCombiner` log errors.
- Updated container images across both nodes: Grafana 11.3.0 → 13.2.0,
  Prometheus 2.55.1 → 3.14.0, node-exporter → v1.12.1, cAdvisor → v0.49.2,
  Portainer 2.33.5 → 2.45.0 (both nodes), Nextcloud 32.0.2 → 32.0.14.
- Began configuring Cloudflare Email Routing + Brevo SMTP relay for
  Nextcloud notifications and guest invites; hit an unresolved 400 error
  saving SMTP settings in Nextcloud. Deferred — logged in
  [`known-issues.md`](./known-issues.md).

## 2025-12-14
- Ran restore-verification testing: stood up a temporary Nextcloud instance
  from a backup snapshot to confirm backups were actually restorable, not
  just present. (This is also roughly when the backup destination
  misconfiguration is believed to have first been introduced — not caught
  until the Aug 2026 triage.)

## Earlier (dates approximate / pre-dates structured tracking)
- Initial homelab build: two-node setup with Nextcloud + MariaDB + Redis on
  the main server, Prometheus + Grafana + Uptime Kuma on the Raspberry Pi.
- Cloudflare Tunnel configured for external access to
  `nextcloud.kitmomedia.com`, avoiding router port forwarding entirely.
- Nightly `rsync`-over-SSH backup script written, targeting the Pi's
  external storage.
