# Code walkthrough — `registry`

What each file does and why, briefly. Deeper reasons: `Dn` in [design-decisions.md](design-decisions.md).
**Written by:** Claude, at the user's request (2026-11-10); the user reviews.

| # | File | Status |
|---|------|--------|
| 1 | `meta/main.yml`, `meta/argument_specs.yml` | done |
| 2 | `.yamllint`, `.ansible-lint`, `.gitignore` | done, copied from `mongodb-cis` |
| 3 | `defaults/main.yml`, `vars/main.yml` | done |
| 4 | `tasks/main.yml`, `install.yml`, `service.yml`, `gc.yml` | done, lint clean, not run on a VM yet |
| 5 | `handlers/main.yml` | done |
| 6 | `templates/registry.container.j2`, `registry-gc.service.j2`, `registry-gc.timer.j2` | done |
| 7 | `requirements.yml`, `VERSION`, `README.md` | done |

---

## How it works in one picture

```text
Ansible writes                         systemd makes                    Podman runs
/etc/containers/systemd/      daemon-reload      registry.service    start     container "registry"
  registry.container   ─────────────────────►  (generated unit)  ──────────►  registry:2.8.3
  (Quadlet file)            (Quadlet generator)                                 listens on :5000 inside

client ──► host :registry_port ──(Podman NAT rule)──► container :5000
                                                      /var/lib/registry  ◄── registry_data_dir on the host (:z)
```

The registry is a container, but systemd supervises it: start at boot, restart on crash, `systemctl` and `journalctl` work as for any service (D1).

---

## 1. `meta/main.yml`: role metadata (no effect on hosts)

| Key | Does | Why |
|-----|------|-----|
| `namespace: bunnywkwk`, `role_name: registry` | Full name `bunnywkwk.registry` | `ansible-pipeline` reads these to run the role; namespace = GitHub owner |
| `min_ansible_version: "2.16"` | Oldest ansible-core allowed | Same as the pipeline's `/opt/ci` (2.16) |
| `platforms: EL 9` | Supported OS | RHEL 9 only (D5) |
| `dependencies: []` | No other roles run first | The jenkins role must not depend on this one; values pass through variables |

### `meta/argument_specs.yml`

Ansible checks every variable's type **before the first task**: `registry_port: "abc"` stops the play with a clear message instead of a broken unit file. It lists the same variables as `defaults/main.yml`.

## 2. `.yamllint`, `.ansible-lint`, `.gitignore`

Copied from `mongodb-cis`: profile `production`, no line-length limit, `true`/`false` only. `.gitignore` skips the `.ansible/` lint cache. The pipeline picks up both lint files from the role root.

## 3. `defaults/main.yml` and `vars/main.yml`

| File | Holds | Rule |
|------|-------|------|
| `defaults/main.yml` | Your choices: image, port, data dir, GC, firewall | Override in `group_vars` |
| `vars/main.yml` | Fixed internals: package list, service name, Quadlet path | Not for users; changing them breaks the role |

| Variable | Default | Why this default |
|----------|---------|------------------|
| `registry_image` | `docker.io/library/registry:2.8.3` | Pinned (D8); fully qualified, so Podman never asks which registry "registry:2" means |
| `registry_port` | `5000` | The registry's usual port; Jenkins uses 8080 on the same VM |
| `registry_data_dir` | `/var/lib/registry` | Every pushed layer lands here: point it at the large disk |
| `registry_gc_enabled` | `false` | GC stops the registry while it runs (D7) |
| `registry_gc_schedule` | `Sun *-*-* 03:00:00` | systemd `OnCalendar` syntax: Sundays 03:00 |
| `registry_manage_firewall` | `true` | Clients on other hosts need the port open |

## 4. Tasks

### `tasks/main.yml`

Imports the three area files in order. `gc.yml` only when `registry_gc_enabled` is `true`. Static `import_tasks`, so `ansible-lint` and `--list-tasks` see every task.

### `tasks/install.yml`

| Task | Does | Why |
|------|------|-----|
| `INSTALL \| Install Podman` | `dnf` installs `podman` | Runs the container, and ships the Quadlet generator that turns our `.container` file into a service. `state: present`, so a rerun changes nothing |

### `tasks/service.yml`

| Task | Does | Why |
|------|------|-----|
| `SERVICE \| Create the data folder` | `registry_data_dir`, `root:root 0700` | Must exist before the container mounts it; only root (the container runs as root) needs it |
| `SERVICE \| Install the registry Quadlet unit` | Writes `/etc/containers/systemd/registry.container`; notifies the restart handler | Changing image, port or data dir changes this file, so the registry restarts only then |
| `SERVICE \| Start the registry` | `daemon_reload: true`, `state: started` | `daemon-reload` runs the Quadlet generator, so `registry.service` exists on the first run. **No `enabled: true`**: systemd refuses to enable generated units; the `[Install]` section of the Quadlet file starts it at boot instead |
| `SERVICE \| Open the registry port` | firewalld `registry_port/tcp`, permanent and immediate | Without `immediate` it opens only after a reload; without `permanent` it closes after a reboot |

### `tasks/gc.yml` (only with `registry_gc_enabled: true`)

| Task | Does | Why |
|------|------|-----|
| `GC \| Install the garbage-collection service and timer` | Writes `registry-gc.service` and `registry-gc.timer` to `/etc/systemd/system/` | A scheduled job in systemd is two files: what to run and when |
| `GC \| Enable the garbage-collection timer` | Enables and starts the **timer** (not the service) | The timer starts the service on schedule |

## 5. `handlers/main.yml`

| Handler | Does | Why |
|---------|------|-----|
| `Restart registry` | `daemon-reload`, then restart `registry.service` | Runs once at the end of the play, only when the Quadlet file changed. Never restarts on a rerun with no change |

## 6. Templates

### `registry.container.j2` (Quadlet)

| Line | Does | Why |
|------|------|-----|
| `[Unit] Wants/After=network-online.target` | Starts after the network is up | The published port needs the network |
| `ContainerName=registry` | Fixed container name | `podman ps`, `podman logs registry` and the GC service use it |
| `Image=` | The pinned image | Pulled on first start |
| `PublishPort=registry_port:5000` | NAT from the host port to 5000 inside | Only the host side changes with `registry_port`; the registry always listens on 5000 inside (D6) |
| `Volume=...:/var/lib/registry:z` | Host data dir mounted into the container | `:z` relabels it `container_file_t` so SELinux lets the container write; lowercase `z` = shared, because the GC container mounts it too |
| `Environment=REGISTRY_STORAGE_DELETE_ENABLED=true` | Allows deleting images through the API | Without it, GC has nothing to remove |
| `[Service] Restart=always` | systemd restarts it if it stops | A crash or `podman stop` does not leave the registry down |
| `[Install] WantedBy=multi-user.target` | Starts at boot | The Quadlet way of "enabled" |

### `registry-gc.service.j2` and `registry-gc.timer.j2`

| File | Does | Why |
|------|------|-----|
| `.service` (`Type=oneshot`) | Stops the registry, runs `registry garbage-collect --delete-untagged` in a throwaway container on the same data dir, starts the registry again (`ExecStopPost`, also after a failed GC) | GC while pushes run can corrupt images |
| `.timer` (`Persistent=true`) | Runs the service at `registry_gc_schedule` | `Persistent` runs a missed slot at the next boot |

## 7. `requirements.yml`, `VERSION`

| File | Does | Why |
|------|------|-----|
| `requirements.yml` | `ansible.posix >=2.2.0,<3.0.0` | The firewalld module; 2.x supports ansible-core 2.16 |
| `VERSION` | `0.1.0` | The pipeline tags `v<VERSION>` after a merged PR (later, when onboarded) |
