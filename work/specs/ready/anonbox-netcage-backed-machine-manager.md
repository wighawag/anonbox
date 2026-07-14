---
title: anonbox — netcage-backed anonymized machine manager (anonctl verb API over netcage)
slug: anonbox-netcage-backed-machine-manager
needsAnswers: true
---

> Launch snapshot — records intent at creation, NOT maintained. Current truth: `docs/adr/` (decisions) + the code; remaining work: `work/tasks/ready/` tasks. (The technical-detail sections below are trimmed by `to-task` once the work is tasked — they move into tasks/ADRs and this spec settles to its durable framing: Problem / Solution / User Stories / Out of Scope.)

<!-- open-questions -->
<!--
  TRANSIENT BLOCK — stripped by the apply rung on full resolution.
  While the spec has unresolved questions blocking autonomous tasking:
    1. Set `needsAnswers: true` in the frontmatter above.
    2. List the questions under the `## Open questions` heading below.
    3. Clear the flag (and let apply strip this block) once they are answered.
  Delete the whole fenced block — markers and all — if the spec launches fully resolved.
-->

## Open questions

These are genuine design forks, deliberately NOT force-resolved during the idea interview. The auto-tasker must refuse to task until a human resolves them and clears `needsAnswers`.

1. **netcage ↔ anonbox version/capability contract.** netcage's jail-intrinsic host-recon binds (`/etc/passwd`, `/etc/machine-id`, `/etc/hostname`/UTS, the LAN-topology `/sys` close) are an in-progress ADR-0013 extension on the netcage side. How does anonbox DETECT whether the netcage on PATH supports these `/etc`-identity machine flags, and how does it DEGRADE when they are absent? Options span: (a) require a minimum netcage version and refuse to run below it (hard floor, the anon-pi `>= 0.12.0` precedent); (b) feature-detect at runtime (probe a `--help`/capability surface or a version string) and compose the flags only when present, running without those binds and SAYING SO in `status`/`verify` output when absent; (c) a hybrid (soft-warn below a floor, hard-refuse below an older floor). Decide the detection mechanism AND the degradation posture (refuse vs run-degraded-and-announce), and how `verify`'s "the `/etc`-identity binds present" account-layer assertion behaves when the running netcage cannot provide them.

2. **Does `rm` remove the Unix account by default?** `rm <name>` removes the machine's home + metadata + kept container. Whether it ALSO deletes the underlying `anonbox-<name>` Unix account is undecided — the anonctl `--purge-account` analogue. Options: (a) `rm` leaves the account and only an explicit `--purge-account` flag deletes it (conservative, matches anonctl's split); (b) `rm` purges the account by default and a flag KEEPS it. Decide the default and the flag name, mirroring anonctl's stance so a user who knows anonctl is not surprised.

3. **Per-machine egress endpoint: require-distinct or allow-sharing?** Each machine can bind its own proxy; anonctl's `<account>@` Tor-isolation-username trick gives unlinkable exits per identity. Decide whether anonbox REQUIRES a distinct egress endpoint per machine (strongest unlinkability, mirrors the per-persona exit) or ALLOWS sharing one endpoint across machines with the documented linkability caveat (mirroring anonctl's share-class boundary). This sets whether the on-disk machine record carries a per-machine endpoint + share-class and how `add`/`status` surface it.

4. **Where does the verify-gated "enter the identity" primitive live for v1?** `use` (interactive shell) and `exec` (one program) are two faces of the same "verify-green-THEN-enter-the-identity" gate; anonctl's `use`/`exec` are the other two faces (its substrate is `setpriv`, anonbox's is `netcage start` + run-inside). Decide whether this shared primitive is FACTORED INTO anoncore now (so anonctl and anonbox share one verify-gated enter core, each supplying its substrate drop) or KEPT IN anonbox for v1 and only lifted into anoncore later once both shapes are proven. This is a founding-ADR-adjacent module-boundary call (the same shared-vs-per-repo tradeoff `anoncore`'s founding ADR pins).

<!-- /open-questions -->

## Problem Statement

You want to do real, ongoing work on the internet without that work being attributable to you or your host. anonctl already solves part of this: it forces one Unix account's egress through an anonymizing proxy at the kernel, fail-closed, and proves it with a leak-test. But anonctl is per-UID only, and its own README's "NOT defended" section is explicit that it structurally CANNOT close the host-recon leak class — a tool running in an anonctl anon account leaks no real IP yet can still fingerprint the host in detail from ordinary unprivileged sources (world-readable `/etc/passwd` real names, `/etc/machine-id`, `/etc/hostname`, other users' processes, host open ports, LAN topology). anonctl NAMES netcage as the different, namespace-strength tool that closes that gap.

So the user needs a tool that gives them named, persistent, anonymized MACHINES — durable workstations they can log into at any time — where egress is proven anonymized (netcage's job) AND the host-recon / on-host-linkability class is closed (a dedicated account + the jail's namespaces). Today the only netcage-backed machine experience is anon-pi, which is pi-specific (and being retired). There is no tool-agnostic, anonctl-API machine manager over netcage.

## Solution

anonbox: a Go CLI that manages named, persistent, anonymized machines over the netcage network-namespace substrate, replicating anonctl's exact verb API so a user who knows anonctl knows anonbox — the only difference being the isolation substrate (netcage netns + namespaces instead of anonctl's per-UID nftables).

A **machine** is three things bound together:

- an **account**: a dedicated host account `anonbox-<name>` (bare `anonbox` as the empty-suffix default), a THIRD non-colliding namespace alongside anonctl's `anon-` and anon-pi's `anonpi-` (so all three tools run on one host). The account exists for ON-HOST UNLINKABILITY only — a mode-700 DAC wall, so mount-source paths and file ownership belong to the machine's account, not the operator's login user (closing the operator-username residual netcage alone can only reduce to a numeric uid). It needs NO nftables/shim/per-UID forcing, because anonymity (egress) is netcage's job, proven by `netcage verify`.
- an **assigned container**: one long-lived, named, KEPT netcage container (no `--rm`), revived on login via `netcage start`. Replaceable underneath the account/home via a rebuild/set-image verb.
- a **mounted home**: the machine's mode-700 home, mounted into the jail as `$HOME` and nothing else (no `/projects` mount — that was pi-specific).

The two load-bearing verbs are verify-gated: `use <name>` runs `netcage verify` bound to the machine's proxy and drops a shell ONLY on green (refuses on red, dropping nothing); `exec <name> <program> [args...]` does the same gate but runs a program with args forwarded verbatim, so no tool needs its own launcher (`anonbox exec <machine> pi ...` runs pi anonymized). anonbox imports `anoncore` for the account/seed/marker/elevation core and owns NONE of anonctl's egress machinery — netcage owns egress; the account is the unlinkability boundary; they compose and neither reimplements the other.

A machine PRESERVES across reboot (account + home + kept container all survive; netcage's `/var/tmp` graphroot is reboot-safe). Unlike anonctl there is no boot-time forcing to re-arm (netcage applies egress fresh on each `start`), so anonbox needs no nftables-service-style boot unit: same durability promise, simpler reboot story.

## User Stories

1. As an operator, I want `anonbox add <name>`, so that a new anonymized machine is provisioned (its dedicated `anonbox-<name>` account created with no sudo/wheel grant, its mode-700 home, its kept netcage container, its metadata record).
2. As an operator, I want `anonbox add` to import NOTHING from my host by default, so that a fresh machine does not carry my credentials/config (a machine that authenticates AS me while hiding my IP defeats the point — the credential-shedding guard is load-bearing).
3. As an operator, I want `anonbox use <name>`, so that I am dropped into a shell inside the machine ONLY after `netcage verify` proves egress is anonymized (green), and refused with nothing dropped on red.
4. As an operator, I want `anonbox exec <name> <program> [args...]`, so that I can run any program in the anonymized machine with args forwarded verbatim, behind the same verify gate — no per-tool launcher needed.
5. As an operator, I want `anonbox use`/`exec` to self-elevate into the `anonbox-<name>` account, so that the machine's files and mount-source paths are owned by that account and not by my login user.
6. As an operator, I want `anonbox list`, so that I can see all my machines.
7. As an operator, I want `anonbox status <name>`, so that I can see a machine's account, home, container, egress endpoint, and whether the netcage `/etc`-identity binds are in effect.
8. As an operator, I want `anonbox verify <name>`, so that the egress proof is DELEGATED to `netcage verify` and anonbox adds only account-layer assertions (right account, home ownership, the `/etc`-identity binds present) — no second egress prover.
9. As an operator, I want `anonbox rm <name>`, so that the machine's home + metadata + kept container are removed (whether the Unix account is also purged is an open question — see Open questions #2).
10. As an operator, I want `anonbox seed-home <name> ...`, so that I can seed a machine's home from a template with the credential-shedding hardening applied (setuid-strip, symlink refusal, mode-700), reusing anoncore's `seedhome`.
11. As an operator, I want `anonbox update`, so that I can update anonbox itself, mirroring anonctl's `update` verb.
12. As an operator, I want a verb that REPLACES a machine's container (rebuild/set-image) from a new image WITHOUT touching the account or home, so that I can reset a machine's rootfs/packages while keeping its durable identity.
13. As an operator, I want my machines to survive a reboot (account + home + kept container all persist), so that a machine is a durable workstation, not a throwaway — with no boot unit required (netcage re-applies egress on each `start`).
14. As an operator, I want anonbox to drive netcage by composing its argv (never touching podman directly), so that netcage stays the sole egress authority and the forced-egress invariant is never weakened by anonbox.
15. As an operator, I want anonbox to compose netcage's `/etc`-identity machine flags WHEN the netcage on PATH supports them, and to tell me clearly when it cannot (degrade-and-announce or refuse — see Open questions #1), so that I am never silently running without the host-recon binds I expect.
16. As an operator on one host, I want anonbox's `anonbox-<name>` accounts to coexist with anonctl's `anon-` and anon-pi's `anonpi-` accounts without collision, so that all the anon* tools run side by side.
17. As an operator, I want the honest residual documented in the README in the netcage/anonctl house style (the shared-kernel/hardware/timing fingerprint — a machine is not a VM — and that the `anonbox-<name>` account is plain Unix DAC: on-host unlinkability, NOT hard containment against host root), so that the machine's fingerprint and its boundary are not surprising.
18. As an operator, I want anonbox to reuse anoncore (not reimplement account provisioning, seedhome, marker, sudoprobe, ui, elevation), so that the security-critical hardening cannot drift between anonctl and anonbox — that is why anoncore was extracted.

### Autonomy notes (the two gate axes)

- **`humanOnly`:** NOT set. The tasking of this spec does not itself require a human to drive it (the security judgement lives in the individual tasks' own gates, which the tasker decides per task from each task's build-nature). Once the open questions are resolved, this spec is straightforwardly agent-taskable.
- **`needsAnswers`: true.** Four genuine design forks (Open questions #1–#4) are unresolved by design. The auto-tasker must refuse to task until a human answers them and clears the flag. Flagging incomplete is correct here: force-resolving these would produce wrongly-cut tasks (e.g. tasking `rm` before its account-purge default is decided).

## Implementation Decisions

Settled at launch (details trimmed into tasks/ADRs at tasking-time):

- **Language & module:** Go (`module github.com/wighawag/anonbox`, `go 1.26`), matching the family. This is OS/exec/account work Go does well, and the account/verify/provision machinery already exists as tested Go in anoncore.
- **Imports anoncore (`github.com/wighawag/anoncore` v0.1.0, published):** the account-hardening half — `provision` (account creation with NO sudo/wheel grant), `seedhome` (setuid-strip, symlink refusal, mode-700), `ui`, `sudoprobe`, `marker`, `account`, `endpoint` as needed. Imports NONE of anonctl's egress machinery (nftables/shim/lanexempt/forcing/systemd): netcage owns egress.
- **Substrate glue is anonbox's own:** composing netcage argv (`run`/`start`/`verify`/`forward`/`ports`, `--allow-direct`, `-v home`, the `/etc`-identity machine flags when supported). anonbox never touches podman. This mirrors how anon-pi drives netcage today.
- **Verb API = anonctl parity:** `add / rm / list / status / verify / use / exec / seed-home / update`, plus a container-replace verb (rebuild/set-image). A user who knows anonctl knows anonbox.
- **`verify` delegates egress to `netcage verify`** and adds only account-layer assertions; it does NOT grow a second egress prover.
- **Account namespace `anonbox-<name>`** (bare `anonbox` default), a third non-colliding namespace alongside `anon-` and `anonpi-`; no per-UID forcing.
- **Root-requiring operations self-elevate** (anonctl's stance, anoncore's `ui` helpers).
- **On-disk machine record / contract:** a per-machine metadata record (account, home path, container name, egress endpoint + share-class) anonbox reads/writes; define it as a SPEC-first on-disk contract (the pattern the family uses so consumers never disagree on layout). Exact shape informed by the resolution of Open questions #3.

## Testing Decisions

- **Acceptance seam:** anonbox's own tests PLUS the delegated `netcage verify`. anonbox does not reimplement an egress prover, so its tests assert the account layer (right account, home ownership, mode-700, the `/etc`-identity binds composed when the netcage supports them, credential-shedding on seed) and that it composes the correct netcage argv; the egress proof itself is netcage's leak-test.
- **Hermetic unit tests, root behind a build tag:** follow anoncore/anonctl's discipline — unit tests run WITHOUT root against an injected `Runner`/fake (no real `useradd`/`setpriv`/`netcage`); the real system-mutating behaviour lives behind an `integration` build tag and SKIPs (not fails) when not root, so a plain `go test ./...` is fully hermetic.
- **Highest seam:** test at the argv-composition + account-assertion seam and at the CLI-verb seam, not by shelling out to a real netcage/podman in unit tests. The netcage feature-detection (Open questions #1) is a natural seam to test against a fake netcage reporting different capability levels.

## Out of Scope

- **Egress forcing / an egress prover.** netcage owns egress and its `verify` proves it; anonbox delegates and never reimplements it. anonctl's nftables/shim/lanexempt/forcing machinery is deliberately NOT imported.
- **A `/projects` per-project mount** (pi-specific; anonbox mounts the home only; host files are mounted per-run via netcage's own `-v`).
- **A boot unit / boot-time re-arm** (unnecessary: netcage applies egress fresh on each `start`; reboot-preservation is just "account + home + kept container still on disk").
- **The netcage-side jail-intrinsic `/etc`-identity binds themselves** (that is netcage's ADR-0013 extension; anonbox only COMPOSES them when available — see Open questions #1).
- **Config-seeding a specific tool's config** (that is anonseed's job; anonbox exposes `seed-home` for generic home seeding and composes with anonseed).
- **Touching podman directly** (anonbox only drives netcage).
- **Hard containment against host root** (the `anonbox-<name>` account is plain Unix DAC — an on-host unlinkability/discoverability boundary, documented in the residual, not defended as a containment wall).

## Further Notes

- **The family (verify live before assuming; sibling repos readable at `../anonctl`, `../anoncore`, `../netcage`):** anonctl (published: per-UID kernel-forced anonymized egress, ships `verify`, already has `exec`) is the API to mirror; anoncore (published, v0.1.0, `github.com/wighawag/anoncore`) is the library to import; netcage (published: forces a container's TCP+DNS egress via network namespace, ships `verify`, drop-in podman-shaped CLI) is the jail to drive; anonseed (being built: Go config-seeder, pi first) composes with anonbox; anon-pi (being retired: its runtime is deleted, its pi config knowledge becomes an anonseed seed) is anonbox's pi-specific predecessor.
- **Naming:** run-together, no hyphen (`anonctl` / `anonbox` / `anonseed` / `anoncore`), anchored on the published name anonctl.
- **The full settled design record** lives in the netcage repo at `work/notes/ideas/netcage-machines-scope-fork.md` (read the TL;DR including its BUILD STATUS banner, and updates 2, 5, 7, 8). anonctl is the API to mirror; anoncore is the library to import; netcage is the jail to drive.
- **A founding ADR** for anonbox is expected once built: the module boundary (what anonbox owns vs what it imports from anoncore vs what it delegates to netcage) is a hard-to-reverse call, referenced by (and referencing) anoncore's founding ADR. Open question #4 (where the verify-gated enter primitive lives) is founding-ADR-adjacent.
