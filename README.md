# TASK 7 — Monitor System Resources Using Netdata

---

## 1. Overview

I installed Netdata to quickly and easily observe system-level and container-level metrics (CPU, memory, disk, network, and Docker containers) in real time. Netdata is lightweight, open-source, and provides a beautiful web dashboard at `http://localhost:19999`.

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

### Output
<img width="1920" height="1080" alt="Screenshot from 2025-10-03 00-09-23" src="https://github.com/user-attachments/assets/0099952c-fbd0-4339-9911-34a624d4443b" />

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
- `/var/run/docker.shock` collect Docker persistant metrics
- `SYS_PTRACE` and `SYS_ADMIN` help Netdata read some system-level metrics when run in a container; these are commonly seen in Netdata Docker examples.
- `--restart unless-stopped` helps Netdata automatically come back after reboots.

---

## 5. Accessing the dashboard

I opened my browser and navigated to:

```
http://localhost:19999
```

Dashboard tips:
- The homepage shows an overview of CPU, memory, disk, and network.
- Use the search box (top-left) to find specific charts (for example, `docker`, `cpu`, `disk`, `processes`).
- There are quick links to view specific charts for CPU, RAM, disk, network, and each Docker container.

### Output
<img width="1920" height="1080" alt="Screenshot from 2025-10-03 00-09-30" src="https://github.com/user-attachments/assets/d6f3dc3a-3a3e-4f51-b188-fecd2d1952ab" />

---

## 6. What I monitored
<img width="1920" height="1080" alt="Screenshot from 2025-10-03 01-31-37" src="https://github.com/user-attachments/assets/6d8755f3-1ee5-43b9-be65-d9126ea310de" />
<img width="1920" height="1080" alt="Screenshot from 2025-10-03 00-27-42" src="https://github.com/user-attachments/assets/167049b4-aba3-4250-a432-327a8954998e" />
<img width="1920" height="1080" alt="Screenshot from 2025-10-03 00-27-53" src="https://github.com/user-attachments/assets/8befbd08-f38d-4c1a-8269-81fe449af5ad" />
<img width="1920" height="1080" alt="Screenshot from 2025-10-03 00-28-37" src="https://github.com/user-attachments/assets/ac9801b9-7245-427c-8def-d2b1425fffb5" />
<img width="1920" height="1080" alt="Screenshot from 2025-10-03 00-43-52" src="https://github.com/user-attachments/assets/cd6e6c9e-af78-4c1b-9f92-2e7c661417f5" />


---

## 7. Alerts and notifications

Netdata ships with pre-configured health checks and alert rules. I explored:
- The **Health** menu on the dashboard to see active alerts and historical alert charts.
- Alert notifications can be routed to many places (Slack, email, PagerDuty, etc.) — configuration is in Netdata's `health_alarm_notify.conf` and the health directory.
- To enable notifications I edit `/etc/netdata/health_alarm_notify.conf` and add my notifier credentials or webhook URLs, then restart Netdata.

---

## 8. Logs and where to find them

Netdata logs and runtime information I used:

- Container logs (docker):
  ```bash
  docker logs -f <container name>
  ```

### Output
<img width="1124" height="512" alt="Screenshot from 2025-10-03 00-27-04" src="https://github.com/user-attachments/assets/d47ebc0f-2d54-48bc-8e0d-945094721e63" />
<img width="1920" height="1080" alt="Screenshot from 2025-10-03 00-51-03" src="https://github.com/user-attachments/assets/b85e7318-e0b4-40fe-b66d-e7c0f2e4cc65" />

---

## 9. Security considerations

I followed these precautions:
- Do not expose port `19999` to the public internet without protection.
- Put Netdata behind an authenticated reverse proxy (Nginx, Traefik) if remote access is needed, or restrict access by firewall or VPN.
- If exposing externally, enable TLS and an auth layer (basic auth or single sign-on).
- Keep Netdata updated: `docker pull netdata/netdata && docker rm -f netdata && docker run ...` to recreate with the new image.

---

## 10. Outcome / What I learned

I used Netdata to get immediate insight into system health and per-container behavior. The live dashboard made it easy to correlate CPU/memory spikes to running containers and processes. The alert system lets me be proactive about issues. Overall, Netdata is a great lightweight monitoring tool that I can use for local development or as the start of an observability stack.

---
