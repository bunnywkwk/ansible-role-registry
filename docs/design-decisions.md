# Design decisions — `registry`

Why the role is built this way. Format: **Decision** → **Why** → **Changed** (if it was).

---

## D1: Container under a Podman Quadlet unit

**Decision:** run `registry:2` as a container that systemd manages through a Quadlet file (`/etc/containers/systemd/registry.container`).

**Why:**
- The brief wants a systemd service. A plain `podman run` survives neither a reboot nor a crash; Quadlet gives a normal `registry.service` (start at boot, `Restart=always`, `journalctl -u registry`).
- Quadlet is part of Podman on RHEL 9: no extra collection, one template.
- Not the native binary: RHEL 9 does not package the registry, and two run modes would double the service code. The brief's "binary or container, chosen by a variable" was dropped for this reason.

## D2: Rootful Podman (runs as root)

**Decision:** the container runs under the system Podman (root), not as a rootless service user.

**Why:** the registry builds nothing and needs no user session. Rootless would need a service user, `loginctl enable-linger` and a user-level Quadlet path, all for a lab service. The rootless trade-off matters for the **jenkins** role (building images), not here.

## D3: Plain HTTP, no TLS (for now)

**Decision:** the registry listens on HTTP.

**Why:** user decision, 2026-11-10, lab only. Clients must list `<host>:<port>` as insecure in `/etc/containers/registries.conf.d/`.

**Changed:** the first version generated a self-signed CA and server cert (`community.crypto`). Removed at the user's request; comes back together with D4.

## D4: No login (for now)

**Decision:** no htpasswd basic auth; anyone who can reach the port can push and pull.

**Why:** user decision, 2026-11-10, lab only. Basic auth over plain HTTP sends the password base64-encoded, readable by anyone on the network, so a login without TLS adds little. The VM is on the libvirt network, reachable only from the workstation and lab VMs.

**Changed:** the first version had an htpasswd user list (`httpd-tools`, check-then-fix per user); judged too much for a lab and removed.

## D5: RHEL 9 and firewalld only

**Decision:** no `ufw` branch, no `ansible_os_family` gating.

**Why:** the target is RHEL 9 only (user, 2026-11-10). One OS needs no branch. `meta/main.yml` declares EL 9.

## D6: No SELinux port label for `registry_port`

**Decision:** the role does not run `semanage port` for a changed port.

**Why:**
- SELinux checks ports at `bind()`, the call a server makes to claim a port. A port label only matters for the **process that binds**.
- Inside the container the registry always binds 5000 (`PublishPort=<host port>:5000`). As far as we know, `container_t` is allowed to bind ports broadly in `container-selinux`, so 5000 passes.
- The host port is a Podman NAT rule: packets are rewritten, nothing confined calls `bind()`, so no label is checked.
- Contrast: `mongod` runs as `mongod_t` on the host and binds its port itself, allowed only `mongod_port_t`. That is why `mongodb8_cis` rule 6.1 labels a non-default port.

**Evidence:** reasoning from how Podman and SELinux work, **not yet tested**. Check V-9 in [test-results.md](test-results.md) proves or disproves it. If an AVC appears, add a `community.general.seport` task.

**What SELinux does need:** the data dir label, done by `:z` on the volume (`container_file_t`).

## D7: Garbage collection optional, off by default

**Decision:** `registry_gc_enabled: false`; when on, a weekly systemd timer stops the registry, runs `garbage-collect --delete-untagged`, and starts it again.

**Why:** pushing over the same tag leaves old layers on disk; GC removes them. GC while pushes run can corrupt images, so the registry stops for its duration, which is why it is off by default.

**Limit:** setting it back to `false` leaves the timer installed (`shortcut:` comment in `tasks/gc.yml`).

## D8: Pinned, fully qualified image

**Decision:** `docker.io/library/registry:2.8.3`.

**Why:** a pinned tag gives the same image on every run (`registry:2` moves). The full name skips Podman's short-name lookup, which can prompt or pick another registry.

## D9: Layout follows `mongodb-cis`

**Decision:** `tasks/main.yml` imports area files; task names `AREA | Do X`; internals in `vars/main.yml`; lint configs copied from `mongodb-cis`; test project like `mongodb8-cis-test`.

**Why:** user request, 2026-11-10: one layout across the user's roles.

## D10: Not onboarded to `ansible-pipeline` yet

**Decision:** no CI build yet, no `ci-container-vars.yml`.

**Why:** user decision, 2026-11-10: onboard when the role is polished, so build resources are not spent on every early push. The role already follows the pipeline's rules (`meta/main.yml` name, `VERSION`, no Jenkinsfile, no `molecule/`).
