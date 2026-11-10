# registry

Ansible role that runs a private container registry (**`registry:2`**) on **RHEL 9**. Podman runs it as a container, and systemd manages it as a normal service through a **Quadlet** unit: started at boot, restarted on a crash, logged in `journalctl -u registry`.

Built for the lab CI host: this registry and the `jenkins` role on one VM, Jenkins pushing the images it builds. The two roles do not depend on each other.

**Lab only for now: plain HTTP and no login.** Anyone who can reach the port can push and pull. Clients must list `<host>:<port>` as insecure (`/etc/containers/registries.conf.d/`). TLS and a login come back together later ([D3, D4](docs/design-decisions.md)).

## What it does

```text
1. INSTALL  podman from the RHEL 9 repos
2. SERVICE  data folder → Quadlet file → registry.service started → port opened in firewalld
3. GC       optional weekly garbage-collection timer (off by default)
```

```text
client ──► host :registry_port ──(Podman NAT)──► container :5000 ──► /var/lib/registry (registry_data_dir)
```

A second run reports `changed=0`. The registry restarts only when its Quadlet file changes.

## Requirements

- RHEL 9 with its repos (`podman`) and access to `docker.io` for the image.
- ansible-core 2.16 or later on the control node.
- Collections: `ansible-galaxy collection install -r requirements.yml` (`ansible.posix`, for firewalld).
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

Every variable, with a one-line explanation, is in [`defaults/main.yml`](defaults/main.yml). Types are checked before the first task ([`meta/argument_specs.yml`](meta/argument_specs.yml)).

## Quick start

```yaml
# requirements.yml
roles:
  - name: registry
    src: https://github.com/bunnywkwk/ansible-role-registry.git
    scm: git
    version: main
collections:
  - name: ansible.posix
    version: ">=2.2.0,<3.0.0"
```

```yaml
# playbooks/site.yml
- name: Set up the CI host registry
  hosts: ci
  become: true
  roles:
    - registry
```

```bash
ansible-galaxy role install -r requirements.yml -p roles
ansible-galaxy collection install -r requirements.yml -p collections
ansible-playbook playbooks/site.yml        # rerun: changed=0
curl http://<host>:5000/v2/                # {} with HTTP 200
```

The lab test project is `ci-host-test/` next to this repo.

## Docs

| File | Read it for |
| --- | --- |
| [docs/code-walkthrough.md](docs/code-walkthrough.md) | What each file and task does, and why |
| [docs/design-decisions.md](docs/design-decisions.md) | Why Quadlet, rootful, no TLS/login, no SELinux port label (`D1`–`D10`) |
| [docs/test-results.md](docs/test-results.md) | Checks that prove the role works: commands, expected and actual results |
| [docs/troubleshooting.md](docs/troubleshooting.md) | Errors met, with cause, fix and prevention (`T-Rn`) |

## Known limits

- Plain HTTP, no login (lab only).
- Setting `registry_gc_enabled` back to `false` leaves the timer: `systemctl disable --now registry-gc.timer`.
- Changing `registry_port` leaves the old port open in firewalld.
