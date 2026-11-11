# Test results — `registry`

Checks that prove the role works, with the exact command and the expected result. **Result** says what actually happened; `not run` means no evidence yet.
Lab playbook: `/home/bunny/cicd-automation/cicd_lab_master` (inventory: `rhel9-ci` = 192.168.122.135, user `frqadmin`).

**Changed 2026-11-11:** the test projects `ci-host-test/` and `k3s-test/` were replaced by one lab playbook, `cicd_lab_master/` (user request). Results dated before that were run from the old projects; their logs moved to `cicd_lab_master/runs/`.

## Static checks (workstation, no VM)

| # | Check | Command | Expected | Result |
|---|-------|---------|----------|--------|
| S-1 | YAML lint | `yamllint .` | no output | ✅ 2026-11-10, yamllint 1.38.0 |
| S-2 | Ansible lint | `ansible-lint --profile production` | `Passed: 0 failure(s)` | ✅ 2026-11-10, ansible-lint 26.11.0 on ansible-core 2.16.20 (`/opt/ci`) |

## Runtime checks (RHEL 9 VM)

Run from `cicd_lab_master/` unless the command says `(VM)`: those run on the VM over SSH.

| # | Check | Command | Expected | Result |
|---|-------|---------|----------|--------|
| V-1 | First run | `ansible-playbook playbooks/main_playbook.yml -J --tags registry` | `failed=0`, `changed` > 0 | not run |
| V-2 | Idempotent | same, rerun | `changed=0` | not run |
| V-3 | Service up | (VM) `systemctl is-active registry` | `active` | not run |
| V-4 | Port claimed | (VM) `sudo ss -tlnp \| grep ':5000'` | one listener on `*:5000` | not run |
| V-5 | API answers | `curl -s -w ' %{http_code}\n' http://192.168.122.135:5000/v2/` | `{} 200` (no login, D4) | not run |
| V-6 | Push and pull | see [below](#v-6-push-and-pull) | push and pull succeed; catalog lists `test/alpine` | not run |
| V-7 | Firewall | (VM) `sudo firewall-cmd --list-ports` | contains `5000/tcp` | not run |
| V-8 | SELinux files | (VM) `ls -Zd /var/lib/registry` | `container_file_t` | not run |
| V-9 | SELinux port (D6) | set `registry_port: 5001` in `sysconfig/group_vars/ci.yml`, run, then V-5 on 5001 and (VM) `sudo ausearch -m AVC -ts recent` | `200` on 5001; `<no matches>` | not run |
| V-10 | Survives reboot | (VM) `sudo reboot`; then V-3 and V-5 | `active`, `200` | not run |
| V-11 | Restart on crash | (VM) `sudo podman kill registry`; wait 10 s; V-3 | `active` again (`Restart=always`) | not run |
| V-12 | GC (optional, D7) | see [below](#v-12-garbage-collection) | timer listed; GC log ends cleanly; registry `active` after | not run |

Undo V-9 afterwards: remove `registry_port: 5001`, rerun (the handler moves the registry back to 5000). Port 5001 stays open in firewalld; close it with `sudo firewall-cmd --permanent --remove-port=5001/tcp && sudo firewall-cmd --reload`.

### V-6: push and pull

On the VM (Podman is there after the role). `--tls-verify=false` because the registry is plain HTTP (D3).

```bash
sudo podman pull docker.io/library/alpine:3.20
sudo podman tag docker.io/library/alpine:3.20 localhost:5000/test/alpine:3.20
sudo podman push --tls-verify=false localhost:5000/test/alpine:3.20
curl -s http://localhost:5000/v2/_catalog                 # {"repositories":["test/alpine"]}
sudo podman rmi localhost:5000/test/alpine:3.20
sudo podman pull --tls-verify=false localhost:5000/test/alpine:3.20
sudo ls /var/lib/registry/docker/registry/v2/repositories  # test
```

Expected: every command exits 0; the layers are on the host in `registry_data_dir`.

### V-12: garbage collection

Set `registry_gc_enabled: true` in `sysconfig/group_vars/ci.yml`, run the playbook, then on the VM:

```bash
systemctl list-timers registry-gc.timer        # next run: Sunday 03:00
sudo systemctl start registry-gc.service       # run it now
journalctl -u registry-gc --no-pager | tail     # garbage-collect output, no error
systemctl is-active registry                   # active again
```

## Evidence

Save each run's output under `cicd_lab_master/runs/` and fill in the Result column with the date and the key numbers.
