---
title: Systemd + Systemctl 
tags: [linux, systemd, systemctl, services, devops, backend]
summary: A concise guide to Linux systemd and systemctl for managing services, including key commands, unit file structure, and practical tips for developers.
aliases:
  - Systemd & Systemctl Quick Start
---

# ⚙️ Linux systemd & systemctl: From Basics to Service Mastery

`systemd` is the **init system and service manager** used by most modern Linux distributions (Ubuntu, Debian, Fedora, Arch, CentOS, etc.).  
It’s what boots your system, starts background services, handles logs, timers, sockets, and shutdowns.

`systemctl` is the command-line interface for controlling `systemd`.

---

## 🧠 1. What systemd Actually Does

When Linux boots, `systemd`:
1. Mounts file systems.  
2. Spawns essential processes.  
3. Starts all enabled services.  
4. Manages dependencies and restarts on failure.

It replaces older systems like `SysVinit` and `upstart`, providing faster parallel startup and fine-grained control.

👉 In short: `systemd` is the **conductor**; services are the **instruments**.

---

## 🧩 2. Key systemctl Commands (Quick Reference)

```bash
# Service management
sudo systemctl start docker
sudo systemctl stop nginx
sudo systemctl restart postgresql
sudo systemctl reload nginx          # reload config without stopping
sudo systemctl status redis          # check active status + logs

# Boot-time behavior
sudo systemctl enable docker         # start automatically at boot
sudo systemctl disable docker        # disable at boot
sudo systemctl is-enabled nginx      # check status

# Logs and info
journalctl -u nginx                  # logs for one service
journalctl -xe                       # detailed system logs
systemctl list-units --type=service  # all active services
systemctl list-timers                # scheduled timers

# System power control
sudo systemctl reboot
sudo systemctl poweroff
````

---

## 🧱 3. Anatomy of a Systemd Unit File

Unit files define how services behave. They live under:

```
/etc/systemd/system/      # custom user units
/lib/systemd/system/      # system packages
```

Example: `/etc/systemd/system/myapp.service`

```ini
[Unit]
Description=My Java Spring Boot Application
After=network.target

[Service]
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/java -jar app.jar
Restart=on-failure
EnvironmentFile=/opt/myapp/.env

[Install]
WantedBy=multi-user.target
```

### Explanation:

* `[Unit]` — dependencies and metadata.
* `[Service]` — what to run, as whom, and how to handle restarts.
* `[Install]` — links service into system targets (boot-time groups).

Reload after editing:

```bash
sudo systemctl daemon-reload
sudo systemctl enable myapp
sudo systemctl start myapp
```

---

## 🧰 4. Common Service States

| Command                   | Meaning                       |
| ------------------------- | ----------------------------- |
| `active (running)`        | Service is up.                |
| `inactive (dead)`         | Stopped.                      |
| `failed`                  | Crashed or exited abnormally. |
| `reloading`               | Reloading config.             |
| `activating/deactivating` | In startup/shutdown phase.    |

---

## 🧩 5. Dependency and Target System

systemd organizes services into **targets** — logical groups like:

* `multi-user.target` → standard multi-user system (normal operation)
* `graphical.target` → with GUI
* `network.target` → networking online
* `default.target` → system’s default startup target

Check default:

```bash
systemctl get-default
```

Change it (e.g., to boot without GUI):

```bash
sudo systemctl set-default multi-user.target
```

---

## ⚡ 6. Journal Logs and Debugging

`journalctl` reads systemd’s binary logs.

```bash
journalctl -u docker.service     # logs for Docker only
journalctl --since "2 hours ago"
journalctl -f                    # follow logs (like tail -f)
journalctl -k                    # kernel logs
```

Filter by boot:

```bash
journalctl -b -1    # previous boot logs
```

---

## 🧰 7. Timers (systemd’s Cron Alternative)

Timers replace `cron` with more flexibility and better logging.

Example: `/etc/systemd/system/backup.timer`

```ini
[Unit]
Description=Nightly DB Backup

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

Then `/etc/systemd/system/backup.service`:

```ini
[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

Enable:

```bash
sudo systemctl enable --now backup.timer
systemctl list-timers
```

---

## 🧩 8. User Services (Per-User systemd)

You can run systemd-managed processes **without root**:

```bash
systemctl --user enable myscript.service
systemctl --user start myscript.service
systemctl --user status myscript.service
```

Useful for background scripts, language servers, or dev daemons.

Config lives under:

```
~/.config/systemd/user/
```

---

## 🧱 9. Service Restart and Recovery Patterns

Add to `[Service]` section:

```ini
Restart=on-failure
RestartSec=5
StartLimitIntervalSec=0
```

Optional advanced options:

* `Restart=always` → restarts even after manual stop.
* `ExecStartPre=/usr/bin/sleep 10` → delay before launch.
* `ExecStopPost=/usr/bin/logger "Service stopped"` → log cleanup.

---

## 🧠 10. Practical Developer Workflow

Most backend tools (Docker, PostgreSQL, Redis, Nginx) are **systemd-managed**.
Here’s the unified pattern for checking and controlling them:

```bash
sudo systemctl status docker postgresql redis nginx
sudo systemctl restart nginx
sudo systemctl enable postgresql redis
```

To verify all are active:

```bash
systemctl list-units --type=service | grep -E 'nginx|redis|docker|postgres'
```

💡 If you use Docker Compose, those containers run *inside* Docker,
but Docker itself is still a systemd service — `systemctl restart docker` affects them all.

---

## 🧩 11. systemd + Development Environments

### JetBrains IDE Integration

* “Before Launch” task: run `sudo systemctl start <service>` automatically.
* Add shell scripts to your Run Configurations for restarting or tailing logs.
* Combine with remote deployment — JetBrains can SSH + reload services post-deploy.

Example remote deploy script:

```bash
scp -r ./app.jar user@server:/opt/myapp/
ssh user@server "sudo systemctl restart myapp"
```

### VS Code Tasks

Add to `.vscode/tasks.json`:

```json
{
  "label": "Restart Nginx",
  "type": "shell",
  "command": "sudo systemctl restart nginx"
}
```

Run directly from VS Code’s command palette.

---

## 🧰 12. Troubleshooting Common Issues

| Problem                  | Fix                                                                     |
| ------------------------ | ----------------------------------------------------------------------- |
| “Unit not found”         | Create a `.service` file under `/etc/systemd/system` and reload daemon. |
| Service doesn’t start    | Check `journalctl -u <service>`.                                        |
| Config edits not applied | Run `sudo systemctl daemon-reload`.                                     |
| Service loops restart    | Review `Restart=` directives or logs.                                   |
| Permission denied        | Ensure correct `User=` and file ownership.                              |

---

## 🧩 13. Real-World Example: Custom API Service

`/etc/systemd/system/backend.service`

```ini
[Unit]
Description=Spring Boot API
After=network.target

[Service]
User=appuser
ExecStart=/usr/bin/java -jar /opt/backend/app.jar
Restart=always
Environment=SPRING_PROFILES_ACTIVE=prod
EnvironmentFile=/opt/backend/.env

[Install]
WantedBy=multi-user.target
```

Commands:

```bash
sudo systemctl daemon-reload
sudo systemctl enable backend
sudo systemctl start backend
sudo systemctl status backend
```

---

## ✅ 14. Summary

* **systemd** boots, manages, restarts, and logs all major services.
* **systemctl** gives you direct command-line control.
* Services are defined via `.service` unit files — easy to write, easy to automate.
* Timers replace cron with more control.
* Understanding this layer makes you a confident operator — not just a developer.

