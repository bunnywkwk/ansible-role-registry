# ansible-role-registry

Private container registry (`registry:2`) on RHEL 9, run by Podman as a systemd service (Quadlet unit).

**Lab only for now: plain HTTP and no login.** Anyone who can reach the port can push and pull. Clients must list `<host>:<port>` as insecure (`/etc/containers/registries.conf.d/`). TLS and a login come back together later.

## Requirements

- RHEL 9 with its repos (`podman`) and access to `docker.io` for the image.
- Collections: `ansible-galaxy collection install -r requirements.yml`.
- firewalld running, unless `registry_manage_firewall: false`.

## Variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `registry_image` | `docker.io/library/registry:2.8.3` | Pinned image |
| `registry_port` | `5000` | Host port, plain HTTP |
| `registry_data_dir` | `/var/lib/registry` | Image storage; put it on the large disk |
| `registry_gc_enabled` | `false` | Weekly garbage-collection timer; the registry stops while it runs |
| `registry_gc_schedule` | `Sun *-*-* 03:00:00` | systemd `OnCalendar` for GC |
| `registry_manage_firewall` | `true` | Open `registry_port` in firewalld |

## Example playbook

```yaml
- hosts: ci
  become: true
  roles:
    - registry
```

## Install

```yaml
# requirements.yml
roles:
  - name: registry
    src: https://github.com/bunnywkwk/ansible-role-registry.git
    scm: git
    version: main
```

## Known limits

- Setting `registry_gc_enabled` back to `false` leaves the timer: `systemctl disable --now registry-gc.timer`.
