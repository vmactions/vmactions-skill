---
name: vmactions-ci
description: Use when a GitHub Actions workflow must build, test, or run code on a platform the GitHub-hosted runners do not offer -- FreeBSD, OpenBSD, NetBSD, DragonFlyBSD, GhostBSD, MidnightBSD, Solaris, OmniOS, OpenIndiana, Tribblix, Haiku, or Android (BlissOS) -- or on a non-x86_64 arch (aarch64, riscv64, powerpc64). Triggers on "test on FreeBSD/OpenBSD/Solaris in CI", "run CI in a BSD VM", "vmactions", "freebsd-vm", a `uses: vmactions/<os>-vm` step, or "my action needs an OS the runner does not have".
---

# Writing CI with vmactions VM actions

## Overview

GitHub-hosted runners only offer Ubuntu, Windows, and macOS. The `vmactions/<os>-vm` actions boot a QEMU VM of another OS on an `ubuntu-latest` runner, sync the checked-out repo into the VM, and run your commands inside it. Use them to add CI on a platform the runner does not provide.

All `*-vm` actions share one identical input interface (they are generated from a single source), so once you know one you know them all -- only the action name, default shell, and package manager differ per OS.

## When to use

- A project must compile/test on a BSD, illumos/Solaris, Haiku, or Android target.
- A project must verify a non-x86_64 arch (`aarch64`, `riscv64`, `powerpc64`).
- Reproducing a platform-specific bug in CI.

**When NOT to use:** if Ubuntu/Windows/macOS runners already cover the target, use a normal matrix -- a VM is far slower. For maintaining the vmactions repos themselves, use the `vmactions-maintenance` skill instead.

## Quick start

The job always `runs-on: ubuntu-latest` (even for non-x86 arches -- QEMU emulates the guest arch). Always pin the major tag `@v1`.

```yaml
name: Test
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    name: Test in FreeBSD
    env:
      MYTOKEN: ${{ secrets.MYTOKEN }}
    steps:
    - uses: actions/checkout@v6
    - name: Test in FreeBSD
      uses: vmactions/freebsd-vm@v1
      with:
        envs: 'MYTOKEN'      # space-separated names of host envs to forward into the VM
        usesh: true          # run `run:` under sh instead of the OS default shell
        prepare: |
          pkg install -y curl
        run: |
          freebsd-version
          uname -a
          ./configure && make && make test
```

`prepare` runs first (install deps as root), then `run` (your CI). The repo is mounted at the **same path** as on the host, `run` starts in that dir, and all `GITHUB_*` plus `CI=true` are forwarded automatically.

**The VM images are minimal base installs -- no language toolchain (go, node, python, gcc, ...) is preinstalled.** Install everything your CI needs in `prepare`. The exact package names differ per OS; copy them from the target repo's README example or that OS's package repo.

**Forwarding a secret takes two parts** -- `envs` only lists *names*, it cannot read `secrets` itself:
1. Bind the secret to a job/step `env:` var on the host: `env: { API_KEY: ${{ secrets.API_KEY }} }`.
2. List that var's name in `envs`: `envs: 'API_KEY'`. Multiple names are space-separated: `envs: 'API_KEY DB_URL CI_TOKEN'`.

Miss step 1 and the variable silently arrives empty inside the VM.

## Available platforms

Action name is `vmactions/<os>-vm@v1`. Confirm the exact supported releases/arches and the right `prepare` package command in the **target repo's README** before writing -- the matrix evolves.

| Action | OS | Default shell (set `usesh: true` for sh) | Typical package install |
|--------|----|------------------------------------------|-------------------------|
| `freebsd-vm`     | FreeBSD     | tcsh (pre-14) | `pkg install -y <p>` |
| `ghostbsd-vm`    | GhostBSD    | tcsh | `pkg install -y <p>` |
| `midnightbsd-vm` | MidnightBSD | tcsh | `mport install <p>` |
| `dragonflybsd-vm`| DragonFlyBSD| csh  | `pkg install -y <p>` |
| `netbsd-vm`      | NetBSD      | csh  | `/usr/sbin/pkg_add -u <p>` |
| `openbsd-vm`     | OpenBSD     | ksh  | `pkg_add <p>` |
| `solaris-vm`     | Solaris     | sh   | `pkg install <p>` / `pkgutil -y -i <p>` |
| `omnios-vm`      | OmniOS      | sh   | `pkg install <p>` |
| `openindiana-vm` | OpenIndiana | sh   | `pkg install <p>` |
| `tribblix-vm`    | Tribblix    | sh   | `zap install <p>` |
| `haiku-vm`       | Haiku       | sh   | `pkgman install -y <p>` |
| `blissos-vm`     | BlissOS (Android) | mksh, no bash | no package manager |

`usesh: true` gives a POSIX `sh` on every OS -- prefer it for portable scripts unless you specifically need the native shell.

## Key inputs

| Input | Purpose | Default |
|-------|---------|---------|
| `run` | CI commands to run in the VM | -- |
| `prepare` | commands run before `run` (install deps as root) | -- |
| `envs` | space-separated host env **names** to forward | none |
| `usesh` | run `run:` under sh instead of the native shell | `false` |
| `release` | pick an OS version, e.g. `"15.0"` | repo default |
| `arch` | `aarch64`, `riscv64`, `powerpc64` (host stays x86 ubuntu) | `x86_64` |
| `sync` | `rsync` (default), `sshfs`, `nfs`, `scp`, or `no` | `rsync` |
| `copyback` | copy files back from VM to host (rsync/scp only) | `true` |
| `mem` | VM memory in MB | `6144` |
| `cpu` | VM cpu cores | all host cores |
| `nat` | host->VM port forwards (YAML map) | none |
| `sync-time` | sync VM clock via NTP | `false` |
| `disable-cache` | disable apt/VM-image caching | `false` |
| `debug-on-error` | on failure, open a web VNC and wait | `false` |
| `custom-shell-name` | name for the custom-shell wrapper (see below) | `<os>` |

## Common patterns

**Multi-step commands inside the VM** -- instead of one big `run`, start the VM once then use `shell: <os> {0}` in later steps. The wrapper auto-`cd`s into `$GITHUB_WORKSPACE`:

```yaml
    - name: Start VM
      uses: vmactions/freebsd-vm@v1
      with: { sync: nfs }
    - name: Build
      shell: freebsd {0}      # = lowercase OS name; override with custom-shell-name
      run: make
    - name: Test
      shell: freebsd {0}
      run: make test
```

**Non-x86 arch:** set `arch:` and keep `runs-on: ubuntu-latest`. `aarch64` works with the default `rsync`; only `riscv64` is restricted to `sync: scp`. Non-x86 guests are fully emulated by QEMU, so they are **much slower** -- compile-heavy jobs (Go, Rust, C++) may run many times slower than native and can hit the default 6h step/job timeout. Set a generous `timeout-minutes` on the job and keep the workload small.

**Port forward (NAT):**
```yaml
      with:
        nat: |
          "8080": "80"
          udp:"8081": "80"
```

## Common mistakes

- **Wrong default shell.** `run` executes under the OS native shell (tcsh on FreeBSD, ksh on OpenBSD, csh elsewhere) unless `usesh: true`. POSIX `sh` scripts fail under tcsh/csh. Set `usesh: true` for portable scripts.
- **`runs-on` set to an arm runner for `arch: aarch64`.** Keep `ubuntu-latest`; the guest arch is emulated by QEMU. `ubuntu-*-arm` runners are much slower.
- **`riscv64` with the default `rsync`/`sshfs`.** riscv64 only supports `sync: scp`.
- **Forgetting `envs`.** Host secrets/env are NOT forwarded unless named in the space-separated `envs` string (only `GITHUB_*` and `CI` are automatic).
- **Guessing the release/arch matrix.** Read the target repo's README table -- not every release supports every arch.
- **Unpinned version.** Use `@v1` (or a full `@v1.x.y`), never an unpinned ref.

## Reference

Each repo's README (e.g. `vmactions/freebsd-vm`) documents the exact release/arch matrix and a runnable example. The project home is https://vmactions.org . For deeper VM control outside CI (local QEMU, SSH, port-forward), see the `anyvm` skill.
