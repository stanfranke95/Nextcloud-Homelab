# Architecture

Full breakdown of both nodes — what runs where, why it's split this way, and
how traffic and data actually move through the system.

## Why two nodes instead of one

Everything could technically run on a single machine. I split it into two for
a couple of reasons:

- **Separation of concerns** — the main server's job is to run Nextcloud and
  nothing else. Monitoring and backups live on a second, cheaper piece of
  hardware (a Raspberry Pi) so they're not competing with Nextcloud for
  resources.
- **Backup independence** — if the main server's drive dies, the backups
  aren't sitting on that same drive. They're on physically separate storage.

The trade-off: both nodes currently sit on the same power circuit and the
same LAN. A single power outage takes the whole stack down at once, and I've
actually lived through that (see [`incident-report.md`](./incident-report.md)).
I know this is a single point of failure — it's a resource-constraint
trade-off I'm accepting for now, not something I overlooked. A UPS for the Pi
is on the list in [`known-issues.md`](./known-issues.md).

## Node 1 — `nextcloud-server`

**Hardware/OS:** Ubuntu 24.04, 1TB SSD

| Service | Version | Port | Purpose |
|---|---|---|---|
| Nextcloud | 32.0.14 | 11000 → 80 | The actual application |
| MariaDB | 10.6 | internal only | Nextcloud's database |
| Redis | 7-alpine | internal only | Caching + session storage |
| cAdvisor | v0.49.2 | 8080 | Per-container resource metrics |
| node-exporter | v1.12.1 | 9100 | Host-level metrics (CPU, disk, RAM) |
| smart-exporter | v0.12.0 | 9902 | Drive health (S.M.A.R.T. data) |
| Portainer | 2.45.0 | 9000 / 9443 | Docker management UI |
| cloudflared | — | — | Cloudflare Tunnel daemon (bare metal, not containerized) |

Everything except `cloudflared` runs as Docker containers managed through
Portainer. `cloudflared` runs directly on the host as a systemd service —
partly a decision I made early on, partly a happy accident that turned out to
have a real benefit: the tunnel stays up even if Docker itself has a problem,
since it doesn't depend on the container runtime at all.

**Why Redis:** Nextcloud works without it, but file locking and session
handling get noticeably faster with a proper cache in front of the database.
Added it deliberately rather than leaving Nextcloud on defaults.

## Node 2 — `raspberrypi-backup`

**Hardware/OS:** Raspberry Pi 4 (4GB), Debian 12, 1TB SSD split into a small
OS partition (~14GB) and a large data partition (~916GB, mounted at
`/mnt/easystore`)

| Service | Version | Port | Purpose |
|---|---|---|---|
| Prometheus | 3.14.0 | 9090 | Scrapes metrics from every exporter on both nodes |
| Grafana | 13.2.0 | 3000 | Dashboards on top of Prometheus's data |
| Uptime Kuma | 1.23.13 | 3001 | Uptime checks + push alerts to my phone |
| pi-cAdvisor | v0.49.2 | 127.0.0.1 only | Container metrics for the Pi itself |
| pi-node-exporter | v1.12.1 | 127.0.0.1 only | Host metrics for the Pi itself |
| Portainer | 2.45.0 | 9000 / 9443 | Docker management UI for this node |

Two of these (pi-cAdvisor, pi-node-exporter) are bound to `127.0.0.1` on
purpose — only Prometheus, running on the same machine, needs to reach them.
No reason to expose them to the LAN. Prometheus, Grafana, and Portainer are
bound so they're reachable from my browser on the network.

The `/mnt/easystore` partition is also where nightly backups from the main
server land — see below.

## How external traffic reaches Nextcloud

No ports are forwarded on my home router. Instead:

```
User's browser
   → nextcloud.kitmomedia.com (DNS resolved via Cloudflare)
   → Cloudflare's network
   → Cloudflare Tunnel (outbound connection initiated by cloudflared on nextcloud-server)
   → localhost:11000 on nextcloud-server
   → Nextcloud container
```

Domain (`kitmomedia.com`) is registered through Namecheap; DNS is managed
through Cloudflare. The tunnel config is local to the server
(`/etc/cloudflared/config.yml`), not managed through Cloudflare's dashboard —
which means the routing rule lives on the box itself, not in the cloud. One
upside I didn't fully appreciate until testing it: because the tunnel makes
an *outbound* connection to Cloudflare, a router reboot or an ISP-assigned IP
change doesn't break anything — it just reconnects. A traditional port-forward
setup would need a dynamic DNS fix for that same scenario.

## How monitoring data flows

Every exporter on the main server (cAdvisor, node-exporter, smart-exporter)
is scraped over the LAN by Prometheus running on the Pi. Grafana then reads
from Prometheus to build dashboards. Uptime Kuma independently polls
Nextcloud's URL (both the external Cloudflare address and the internal LAN
address, checked separately) so I can tell whether an outage is the tunnel's
fault or the server's fault.

## How backups work

A cron job on the main server runs nightly at 1:30 AM:

1. Dump the MariaDB database from inside the container (`mysqldump`)
2. Put Nextcloud into maintenance mode briefly, to keep the filesystem
   consistent during the copy
3. `rsync` the Nextcloud app/config directory to the Pi over SSH
4. `rsync` the actual user data directory to the Pi over SSH
5. Copy the database dump over as well
6. Take Nextcloud back out of maintenance mode (this step runs even if an
   earlier step fails, via a trap in the script)

The connection to the Pi uses a dedicated `backupuser` account rather than my
personal login — that account can only write to the backup directory, nothing
else on the Pi. If that SSH key were ever compromised, the blast radius is
limited to backup storage, not the whole Pi.

## Future storage expansion

I have two HGST Ultrastar He8 8TB enterprise drives (ex-datacenter, still
healthy) sitting unused, earmarked for expanding storage on both nodes once
current documentation and triage work is caught up. Not installed yet — see
[`known-issues.md`](./known-issues.md).
