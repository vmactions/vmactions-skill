# vmactions-ci skill

An agent **skill** that helps you write GitHub Actions CI for platforms the
GitHub-hosted runners do not offer -- FreeBSD, OpenBSD, NetBSD, DragonFlyBSD,
GhostBSD, MidnightBSD, Solaris, OmniOS, OpenIndiana, Tribblix, Haiku, and
Android (BlissOS) -- and for non-x86_64 architectures (aarch64, riscv64,
powerpc64), using the [vmactions](https://vmactions.org) `*-vm` actions.

When this skill is active, your coding agent (Claude Code, Codex, Copilot CLI,
Gemini CLI, etc.) knows the action names, the full input interface, per-OS
default shells and package managers, and the common footguns -- so it can write
a correct `.github/workflows/*.yml` for you instead of guessing.

The skill itself is [`SKILL.md`](SKILL.md).

## Install

This repo IS the skill (its `SKILL.md` is at the root). Put it where your agent
runtime discovers skills.

**Claude Code** -- personal skills live in `~/.claude/skills/<name>/`:

```sh
git clone https://github.com/vmactions/vmactions-skill ~/.claude/skills/vmactions-ci
```

(or copy this folder there). The directory name does not matter; the skill is
identified by the `name:` field in `SKILL.md`.

**Codex / Copilot CLI / Gemini CLI** -- these also recognize the cross-runtime
path `~/.agents/skills/<name>/`:

```sh
git clone https://github.com/vmactions/vmactions-skill ~/.agents/skills/vmactions-ci
```

Restart the agent (or start a new session) so it picks up the new skill.

## Use

Just describe what you want in plain language -- the skill triggers on its own
when the request matches:

> Add CI that runs my test suite on OpenBSD.

> I need to check this builds on FreeBSD aarch64.

> Set up a GitHub workflow to reproduce this bug on NetBSD.

The agent loads the skill and produces the workflow. You can also invoke it
explicitly in Claude Code with `/vmactions-ci`.

## What it produces

A ready-to-commit workflow. For example, "run `go test` on OpenBSD":

```yaml
name: Test
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    env:
      API_KEY: ${{ secrets.API_KEY }}
    steps:
      - uses: actions/checkout@v6
      - name: Test in OpenBSD
        uses: vmactions/openbsd-vm@v1
        with:
          envs: 'API_KEY'
          usesh: true
          prepare: |
            pkg_add go
          run: |
            go test ./...
```

## See also

- All supported VMs and releases: <https://vmactions.org>
- Powered by [AnyVM.org](https://anyvm.org)
- Each platform's exact release/arch matrix lives in its own repo, e.g.
  [`vmactions/freebsd-vm`](https://github.com/vmactions/freebsd-vm).
