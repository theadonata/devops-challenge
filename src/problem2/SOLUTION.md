# Problem 2: Diagnose Me Doctor

**Scenario:** Ubuntu 24.04 VM, 64GB disk, running only NGINX as a load balancer. Monitoring shows disk usage stuck at 99%.

## How I'd troubleshoot it

1. Check which filesystem is actually full: `df -hT`
2. If it's nearly 100%, truncate the active NGINX logs: `truncate -s 0 /var/log/nginx/access.log /var/log/nginx/error.log`
3. Find out what's taking the space: `du -xhd1 / | sort -rh` or `ncdu -x /`
4. Once I know which of the below it is, apply the matching recovery step.

## Causes I'd expect, and what to do for each

**1. NGINX logs never got rotated**
- Why: no logrotate config, or logrotate exists but isn't running.
- Impact: grows with every request the LB handles. Eventually NGINX can't write logs and starts failing to serve traffic.
- Recovery: add/fix the logrotate config, run `logrotate -f /etc/logrotate.d/nginx` now. Long term, ship logs off the box instead of keeping them local.

**2. Log file deleted but NGINX still holds it open**
- Why: someone deleted the log file directly, so the disk space is still reserved even though the file is gone.
- Impact: `df` shows full, `du` doesn't show why — confusing to debug if you don't know this pattern.
- Recovery: `nginx -s reopen` tells NGINX to reopen its log files, freeing the old space instantly. No downtime.

**3. Traffic spike or attack flooding the logs**
- Why: a burst of requests (real spike, bot traffic, or attack) is writing way more log lines than normal.
- Impact: same symptom as #1, but rotating logs alone won't fix it — it just refills.
- Recovery: rate-limit or block the source in NGINX (`limit_req`, `deny`) or upstream firewall, then fix log rotation so this degrades safely next time instead of filling the disk.

**4. No alerting before it got this bad**
- Why: nothing paged anyone at 70-80% usage, so it was only caught at 99%.
- Impact: not a root cause by itself, but it's why this became an emergency instead of a routine cleanup.
- Recovery: add disk usage alerts (e.g. warn at 70%, page at 85%), especially watching `/var/log` on this box since that's where the risk concentrates.
