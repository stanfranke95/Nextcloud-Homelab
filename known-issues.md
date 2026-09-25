# Known Issues & Roadmap

An honest list of what's imperfect about this setup right now, why it's not
fixed yet, and what the plan is. I'd rather show that I know exactly where
the weak points are than pretend everything's perfect — a system with no
documented known issues either doesn't have any (unlikely) or nobody's looked
closely enough to find them.

## Security / hardening

- **Nextcloud is running an end-of-life version.** Currently on 32.0.14;
  version 32 reached end-of-life in September 2026, meaning it no longer
  receives security patches. Upgrade to 33.x is planned — deliberately not
  done immediately after finding it, so it can be scheduled and tested
  properly rather than rushed on a system someone actively depends on.
- **Grafana's admin credentials are stored in plaintext** in the
  `docker-compose.yml` environment variables. Should be moved to a `.env`
  file (excluded from git) or Docker secrets.
- **Portainer uses the same login credentials on both nodes.** Convenient,
  but if one is ever compromised, so is the other. Needs unique credentials
  per node.
- **No external monitoring.** Uptime Kuma monitors everything else, but
  nothing monitors Uptime Kuma itself — if the Pi goes down, so does my only
  alerting, with no fallback. A free tier of an external service (e.g. a
  third-party uptime checker that isn't running on my own hardware) would
  close this gap.

## Reliability

- **Prometheus and Grafana bind to a static LAN IP instead of `0.0.0.0`.**
  This directly caused the boot-order race condition documented in
  [`incident-report.md`](./incident-report.md) — if the Pi comes up before
  the router finishes reassigning DHCP leases after a power event, these
  containers silently lose their port mappings. Low-effort fix, just hasn't
  been done yet.
- **No UPS on either node.** Both nodes share the same power circuit, so a
  single outage takes the whole stack down at once — as it already has. A
  small UPS on the Pi at minimum would let it shut down gracefully instead of
  losing power mid-write, and would keep Uptime Kuma alive long enough to
  actually alert me during an outage instead of going dark with everything
  else.
- **No notification on backup success or failure.** The incident in
  [`incident-report.md`](./incident-report.md) went undetected for weeks
  partly because nothing pushed an alert when the backup failed. A simple
  Pushover notification on the script's exit status would catch this
  immediately next time instead of relying on me remembering to check
  manually.

## Incomplete work

- **Email sending (SMTP) is not fully configured.** Cloudflare Email Routing
  is set up and working for `noreply@kitmomedia.com`, and a Brevo account is
  created with the domain authenticated — but saving the SMTP config in
  Nextcloud's admin panel consistently fails with a 400 error that I haven't
  root-caused yet. Deferred rather than rabbit-holed on; email isn't blocking
  anything else right now.
- **Guest accounts for external collaborators.** My partner wants to be able
  to share files and collaborate with coworkers through Nextcloud. This needs
  the Guests app plus working email (guest invites are sent by email), so
  it's blocked on the item above.

## Planned expansion

- **Two unused 8TB enterprise HDDs** (HGST Ultrastar He8, previously
  datacenter drives) are sitting ready for when current storage needs grow.
  Not installed yet — want to run a S.M.A.R.T. health check on them first
  since they're used drives, and the Pi will need a USB-to-SATA enclosure
  since it has no internal drive bay.
- **Active Directory / Windows Server lab.** Separate from this homelab, but
  related to closing a skills gap — most entry-level IT roles lean heavily on
  Windows/AD, which this Linux-based homelab doesn't touch at all. Planned as
  a VirtualBox or Hyper-V lab using Microsoft's free evaluation license.

## Explicitly accepted trade-offs (not bugs)

- **Both nodes share power and LAN**, meaning there's a single point of
  failure for the whole stack. This is a known consequence of running a
  two-node homelab on consumer hardware in one location, not something I
  overlooked. Documented here so it's clear it's a conscious call, not a gap
  in understanding.
