# Incident Reports

A running log of real incidents on this homelab — what broke, how I found it,
what the actual root cause was, and what changed as a result. New entries get
added to the top as things come up. This isn't a hypothetical exercise; every
entry here is something that genuinely happened on a system someone
depends on.

---

## Incident #1 — Pi Service Outage & Silent Backup Failure

**Date of incident:** August 23–28, 2026
**Date resolved:** August 28, 2026
**Severity:** High (backup failure) / Medium (service outage)

### Summary

A power outage on August 23 took down services on the Raspberry Pi node.
While recovering from that outage four days later, I discovered a second,
unrelated problem that had been silently running in the background for
weeks: the nightly backup job had been failing every single night without
ever raising an error, and had filled the Pi's OS drive to 100% in the
process. Both issues are documented here together because they were found
and fixed in the same investigation, and because the second one is the more
important lesson of the two.

### Timeline

| Date | Event |
|---|---|
| Aug 23 | Power outage at home. Pi loses power along with the main server. |
| Aug 23, 4:10 AM | Nextcloud's internal cron job fails with a database connection error — a symptom of the outage, not a separate problem. |
| Aug 27 | I check the Pi's web-based services (Portainer, Uptime Kuma) and get `ERR_CONNECTION_REFUSED` on all of them. |
| Aug 27 | SSH into the Pi directly. `docker ps -a` shows Prometheus, Grafana, Uptime Kuma, and Portainer all `Exited`. cAdvisor and node-exporter are still running. |
| Aug 27–28 | Root-cause both the outage and a related but distinct pre-existing bug (below), fix and verify everything. |
| Aug 28 | While verifying backup health as part of the same session, discover the nightly backup script has been failing every night for weeks. |
| Aug 28 | Fix, verify, and clean up the backup issue. |

### Sub-incident A: Pi services down after power outage

**Root cause — two separate issues, not one:**

1. **Missing restart policy on Portainer.** Portainer wasn't part of a
   Docker Compose stack — it had been started manually at some point with a
   `docker run` command that didn't include `--restart unless-stopped`. Every
   other service that had this flag came back up on its own after power was
   restored; Portainer didn't.

2. **A boot-order race condition.** Prometheus and Grafana's compose file
   bound their ports to the Pi's static LAN IP (e.g. `10.0.0.217:9090:9090`)
   rather than `0.0.0.0`. On a normal restart this is fine — the Pi already
   has that IP. But after a power outage, the Pi's network stack can come up
   *before* the router finishes rebooting and re-issuing DHCP leases. Docker
   tried to bind to an IP that didn't exist on the interface yet, so it
   silently started the containers with no port mapping at all — no error,
   just no way to reach them. cAdvisor and node-exporter were unaffected
   because they're bound to `127.0.0.1`, which is always available
   immediately regardless of DHCP timing.

**Resolution:**
- Recreated Portainer manually with the correct `--restart unless-stopped`
  flag and explicit port mappings, preserving its existing data volume.
- Redeployed the Prometheus/Grafana stack through Portainer once the Pi had
  its IP back, which fixed the immediate problem. The proper long-term fix
  — changing those bindings to `0.0.0.0` so this can't recur — is tracked in
  [`known-issues.md`](./known-issues.md).

### Sub-incident B: Silent backup failure (the more important one)

This is the one that actually mattered. The Pi services being down for four
days was annoying but low-stakes — nobody was relying on Grafana dashboards
in real time. A backup that *looks* like it's working but isn't is a much
bigger problem, because you don't find out until the day you actually need
it.

**Detection:** While verifying general system health after the outage, I
checked whether the nightly backup cron job had run recently. The log showed
it had run every night — but every single run had failed with the same
error:

```
rsync: [receiver] write failed on ".../Photos/[filename].MOV": No space left on device (28)
```

**Investigation:** `df -h` on the Pi showed the OS drive (`/dev/sda2`, 14GB)
at 100% capacity, while the large external storage partition
(`/mnt/easystore`, ~916GB) had 865GB free. The backup script's destination
variable (`PI_BASE`) was pointed at `/srv/backups/nextcloud` — a path on the
small OS drive — instead of `/mnt/easystore/backups/nextcloud`, the intended
location on the large partition. This had apparently been misconfigured
since the backup was first set up; it worked initially because the backup
was small enough to fit, and only started failing once the amount of data
being backed up grew past the OS drive's capacity.

**Impact:** Every nightly backup had been failing silently for weeks. No
error was surfaced anywhere I would normally look — the cron job "ran," it
just didn't complete. If the main server's drive had failed during that
window, the most recent usable backup would have been whatever predated the
misconfiguration — potentially weeks of data loss.

**Resolution:**
1. Corrected `PI_BASE` in the backup script to point at
   `/mnt/easystore/backups/nextcloud`.
2. Fixed a second, related problem this exposed: the destination directory
   was owned by my personal user, not `backupuser` (the restricted account
   the script connects as), so even after correcting the path the first
   retry failed with a permissions error. Fixed ownership with `chown -R`.
3. Ran the backup manually rather than waiting for the next scheduled run,
   and watched it complete successfully end to end.
4. Specifically verified that the large file responsible for originally
   filling the drive had made it into the new backup location, rather than
   just trusting a "completed" message.
5. Deleted the 8.7GB of incomplete backup data left behind on the OS drive,
   bringing it back down to a healthy ~32% used.

### Lessons learned

- **A script running without errors is not the same as a script doing its
  job.** The backup "ran" every night. It just didn't do what it was
  supposed to. I don't think I'd have caught this without deliberately going
  and checking, rather than trusting that no alert meant no problem.
- **Restart policies need to be audited across every container, not assumed.**
  I'd set `restart: unless-stopped` on most things when I first built this,
  but Portainer had been created outside that process and never got it.
- **A static IP is not the same guarantee as `127.0.0.1` or `0.0.0.0`** when
  it comes to what's available at container-start time, especially right
  after a power event. This is now something I check specifically when
  reviewing any new compose file.
- Both issues are now tracked with concrete fixes in
  [`known-issues.md`](./known-issues.md) rather than just "fixed and
  forgotten."

---

## Incident #2 — *(next one goes here)*

*No further incidents logged yet as of this writing. When something breaks,
it gets documented here — that's the point of keeping this as a running log
instead of a one-time writeup.*
