# TASK 7 — Monitor System Resources Using Netdata

> **Author / voice:** I wrote this guide as a first-person walkthrough of how I installed and used Netdata to monitor system and application performance using Docker.

---

## 1. Overview

I installed Netdata to quickly and easily observe system-level and container-level metrics (CPU, memory, disk, network, and Docker containers) in real time. Netdata is lightweight, open-source, and provides a beautiful web dashboard at `http://localhost:19999`.

By the end of this guide I captured the dashboard, explored charts and alerts, and inspected Netdata logs.

---

## 2. Prerequisites

Before I started, I made sure I had:

- Docker installed and running on my machine. (Test with `docker --version` and `docker run hello-world`.)
- At least one container or workload running (so I could see container metrics).
- Basic familiarity with the terminal / CLI.

---

## 3. Quick start (single Docker command)

The fastest way I used was the one-line Docker run:

```bash
docker run -d --name=netdata -p 19999:19999 netdata/netdata
```

This starts Netdata and exposes the dashboard on port `19999`. I visited:

```
http://localhost:19999
```

If you're running Docker on a remote host, replace `localhost` with the host IP (and ensure port `19999` is reachable from your browser).

---

## 4. Recommended Docker run (persistent, safer)

For production or longer-term use I prefer to add persistent volumes and a few capability flags so Netdata collects data across restarts and can access host metrics:

```bash
docker run -d   --name=netdata   -p 19999:19999   -v /etc/netdata:/etc/netdata:ro   -v /var/lib/netdata:/var/lib/netdata   -v /var/log/netdata:/var/log/netdata   -v /var/cache/netdata:/var/cache/netdata   --cap-add SYS_PTRACE   --cap-add SYS_ADMIN   --security-opt apparmor=unconfined   --restart unless-stopped   netdata/netdata
```

Notes on the flags and mounts I used:
- `/etc/netdata` stores configuration (read-only mount recommended if you want to manage config on host).
- `/var/lib/netdata` persists collected metrics/history.
- `/var/log/netdata` stores Netdata logs (useful for troubleshooting).
- `/var/cache/netdata` (optional) for caches.
- `SYS_PTRACE` and `SYS_ADMIN` help Netdata read some system-level metrics when run in a container; these are commonly seen in Netdata Docker examples.
- `--restart unless-stopped` helps Netdata automatically come back after reboots.

---

## 5. Docker Compose example

I sometimes prefer `docker-compose.yml`. Here’s a basic example I used:

```yaml
version: "3.8"
services:
  netdata:
    image: netdata/netdata:latest
    container_name: netdata
    ports:
      - "19999:19999"
    volumes:
      - /etc/netdata:/etc/netdata:ro
      - /var/lib/netdata:/var/lib/netdata
      - /var/log/netdata:/var/log/netdata
      - /var/cache/netdata:/var/cache/netdata
    cap_add:
      - SYS_PTRACE
      - SYS_ADMIN
    security_opt:
      - apparmor:unconfined
    restart: unless-stopped
```

Start with:

```bash
docker compose up -d
```

---

## 6. Accessing the dashboard

I opened my browser and navigated to:

```
http://localhost:19999
```

Dashboard tips:
- The homepage shows an overview of CPU, memory, disk, and network.
- Use the search box (top-left) to find specific charts (for example, `docker`, `cpu`, `disk`, `processes`).
- There are quick links to view specific charts for CPU, RAM, disk, network, and each Docker container.

---

## 7. What I monitored

I verified these metric groups:

- **CPU** — per-core utilization, interrupts, load average.
- **Memory** — used, cached, buffers, swap usage and pressure.
- **Disk** — IOPS, throughput, latency, per-device utilization, and disk health indicators (where available).
- **Network** — packets, errors, throughput per interface.
- **Docker containers** — CPU, memory, disk I/O, network per container (Netdata auto-detects Docker stats with default setups).
- **Processes** — top CPU or memory consumers, process-level charts.
- **System services / apps** — if I enabled specific collectors (MySQL, NGINX, Redis, Postgres, etc.), I could view app-specific charts.

---

## 8. Enabling/Checking Docker container metrics

Netdata typically auto-detects Docker metrics. If I didn't see container metrics, I checked:
- Docker daemon is running.
- The Netdata container had permission to access container metrics (cgroups) — the `SYS_ADMIN` and `SYS_PTRACE` caps help.
- I made sure `/sys/fs/cgroup` is accessible to Netdata (the default Docker image handles this in most environments).

If necessary, I bind-mounted cgroup paths to the Netdata container (advanced):

```bash
-v /sys/fs/cgroup:/host/sys/fs/cgroup:ro -v /proc:/host/proc:ro
```

Then, inside Netdata, container charts appear under "Apps (containers)" or by searching for the container name.

---

## 9. Alerts and notifications

Netdata ships with pre-configured health checks and alert rules. I explored:
- The **Health** menu on the dashboard to see active alerts and historical alert charts.
- Alert notifications can be routed to many places (Slack, email, PagerDuty, etc.) — configuration is in Netdata's `health_alarm_notify.conf` and the health directory.
- To enable notifications I edit `/etc/netdata/health_alarm_notify.conf` and add my notifier credentials or webhook URLs, then restart Netdata.

---

## 10. Logs and where to find them

Netdata logs and runtime information I used:

- Container logs (docker):
  ```bash
  docker logs -f netdata
  ```

- Netdata log files (inside host paths I mounted):
  - `/var/log/netdata/error.log`
  - `/var/log/netdata/netdata.log`
  - `/var/log/netdata/access.log` (if enabled)

- If I used the single-command run (no mounts), logs are visible from `docker logs netdata`.

---

## 11. Taking the required screenshot (how I captured dashboard & metrics)

The task asked for a screenshot of the dashboard and running metrics. I captured it by:
1. Opening `http://localhost:19999` in my browser.
2. Navigating to a view with dashboards showing CPU, memory, and container metrics.
3. Using my OS screenshot tool (or browser's developer screenshot) to capture the visible charts. For example:
   - On Linux: `gnome-screenshot` or `PrintScreen` key, or `scrot`.
   - On macOS: `Shift + Cmd + 4` and drag to select.
   - On Windows: `Win + Shift + S` to use Snip & Sketch.
4. Saving the screenshot as `netdata-dashboard.png`.

If you want a command-line screenshot (from a headless server with a browser), I mention two approaches:
- Use a headless browser tool like `puppeteer` or `playwright` to load the page and take a screenshot programmatically.
- Use `cutycapt` or `wkhtmltoimage` (if the server can render) — note that dynamic charts may need a JS-enabled renderer.

---

## 12. Security considerations

I followed these precautions:
- Do not expose port `19999` to the public internet without protection.
- Put Netdata behind an authenticated reverse proxy (Nginx, Traefik) if remote access is needed, or restrict access by firewall or VPN.
- If exposing externally, enable TLS and an auth layer (basic auth or single sign-on).
- Keep Netdata updated: `docker pull netdata/netdata && docker rm -f netdata && docker run ...` to recreate with the new image.

---

## 13. Troubleshooting tips (what I did when things went wrong)

- **No metrics visible for containers**: ensured Docker stats were available; restarted Netdata with appropriate caps and mounts.
- **Missing charts or collectors**: check Netdata's `stream.conf` and `netdata.conf` for disabled modules.
- **High resource use by Netdata**: ensure collection intervals are reasonable; Netdata is lightweight but you can tune collection frequency in `netdata.conf`.
- **Logs show permission errors**: verify host mount permissions (UID/GID) and adjust with `:ro` or proper ownership.

---

## 14. Bonus — sample commands I used

```bash
# Quick check if Netdata container is running
docker ps | grep netdata

# Follow Netdata logs
docker logs -f netdata

# Recreate Netdata after pulling latest image
docker pull netdata/netdata
docker rm -f netdata
# (re-run the docker run command used earlier)

# See mounted volumes on the running container
docker inspect netdata --format '{{ json .Mounts }}' | jq
```

---

## 15. Outcome / What I learned

I used Netdata to get immediate insight into system health and per-container behavior. The live dashboard made it easy to correlate CPU/memory spikes to running containers and processes. The alert system lets me be proactive about issues. Overall, Netdata is a great lightweight monitoring tool that I can use for local development or as the start of an observability stack.

---

## 16. Files I created while following this task

- `README_Netdata.md` (this file)
- `docker-compose.yml` (optional; example in the README)
- `netdata-dashboard.png` (my screenshot — create this locally by following the screenshot steps above)

---

If you'd like, I can:
- Add a `docker-compose.yml` file (I included the example above) as a downloadable file.
- Generate a ready-to-run script (`run-netdata.sh`) that uses the recommended Docker command.
- Help you with a reverse proxy config (Nginx) to secure Netdata.

---

**End of README — I wrote this from a first-person perspective to match the task brief.**
