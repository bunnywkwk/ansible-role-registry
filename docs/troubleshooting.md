# Troubleshooting log: `registry`

Errors hit while building and testing this role, with the exact message, cause and fix.
Newest last. Format: **Symptom** (exact output) → **Cause** → **Fix** → **Prevent**.

---

## T-R1: `Could not find the requested service registry: host` (2026-11-10)

**Symptom** (first run on `rhel9-ci`, commit `6b1e62f`)
```
TASK [registry : SERVICE | Install the registry Quadlet unit]
changed: [rhel9-ci]
TASK [registry : SERVICE | Start the registry]
fatal: [rhel9-ci]: FAILED! => {"changed": false, "msg": "Could not find the requested service registry: host"}
```

**Cause:** `insecure=true` was added to the `[Container]` section of `templates/registry.container.j2`. It is not a Quadlet key. At `daemon-reload` the Quadlet generator rejects a file with an unknown key and generates **no** `registry.service`, so systemd has no such unit. "Insecure" is a setting for **clients** (`registries.conf.d` on the host that pushes), not for the registry, which already serves plain HTTP.

**Fix:** removed the line. Plain-HTTP trust for the pushing side is the jenkins role's `jenkins_insecure_registries`.

**Prevent:** after changing the template, check it on the VM with the generator's dry run, which prints the exact rejected key:
```bash
sudo /usr/libexec/podman/quadlet -dryrun
```
