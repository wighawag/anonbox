# CONTEXT — anonbox domain language

The domain glossary for `anonbox`. Agents and skills use THIS vocabulary when naming modules, tests, and discussing the system. Architectural rationale lives in `docs/adr/` (decisions); product framing lives in `work/specs/`.

## What anonbox is

anonbox is a Go CLI that manages named, persistent, anonymized "machines" over the netcage network-namespace substrate. A machine is a dedicated host account (`anonbox-<name>`) + one long-lived, kept netcage container + a mode-700 mounted home; you log into it (`use`) or run a program in it (`exec`) only after egress is proven anonymized by `netcage verify`. It is the netcage-backed sibling of anonctl: it replicates anonctl's verb API (`add / rm / list / status / verify / use / exec / seed-home / update`) over netcage instead of anonctl's per-UID nftables, and closes the host-recon leak class anonctl structurally cannot (anonctl's own README "NOT defended" section names that gap and points at netcage as the different tool needed). It imports `anoncore` for the account/seed/marker/elevation core and owns NONE of anonctl's egress machinery (netcage owns egress). Part of the `anonctl / netcage / anonseed / anoncore` family (run-together naming, no hyphen, anchored on the published name anonctl).

## Core domain terms

- **machine** — the core unit: an account + an assigned container + a mounted home, a durable anonymized workstation you log into at any time. The home is the durable identity; the container is resettable convenience state (rootfs/packages). A machine PRESERVES across reboot.
- **`anonbox-<name>` account** — the dedicated host account backing a machine (bare `anonbox` is the empty-suffix default). A THIRD non-colliding account namespace alongside anonctl's `anon-` and anon-pi's `anonpi-` (all three coexist on one host). It exists for ON-HOST UNLINKABILITY only (a mode-700 DAC wall; mount-source paths and file ownership belong to the machine's account, not the operator's login user, which closes the operator-username residual netcage alone can only reduce to a numeric uid). It needs NO nftables/shim/per-UID forcing: anonymity (egress) is netcage's job, proven by `netcage verify`.
- **kept container** — each machine owns one long-lived, named, KEPT netcage container (netcage's durable-box lifecycle, no `--rm`), revived on login via `netcage start`. A verb REPLACES the container (rebuild/set-image) from a new image without touching the account or home.
- **home** — the machine's mode-700 HOME, mounted into the jail as `$HOME` and nothing else. anonbox is tool-agnostic and does NOT adopt a per-project `/projects` mount (that was a pi-specific concept). Host files are mounted per-run via netcage's own `-v` if wanted.
- **`use <name>`** — the interactive login verb: run `netcage verify` (bound to the machine's proxy) and drop a shell inside the machine ONLY on green; refuse on red, dropping nothing (the anonctl safety gate). Self-elevates into the `anonbox-<name>` account. Mechanically `netcage start` the machine's container + drop a shell inside, gated by verify.
- **`exec <name> <program> [args...]`** — the generic launcher verb: same verify gate, but run a program (args forwarded verbatim) instead of an interactive shell. Means no tool needs its own launcher (`anonbox exec <machine> pi ...` runs pi anonymized). Mirrors the `exec` verb anonctl already has.
- **verify gate** — the load-bearing "green then enter, refuse on red" safety gate shared by `use` and `exec`. anonbox's own `verify` verb DELEGATES the egress proof to `netcage verify` and adds only account-layer assertions (right account, home ownership, the `/etc`-identity binds present); it does NOT reimplement an egress prover.
- **credential-shedding guard** — a load-bearing constraint: a persistent machine home must NOT default to importing host credentials/config (a machine carrying the operator's tokens stops the IP leak while still authenticating AS the operator, defeating the point). A fresh machine imports nothing. Enforced via anoncore's seedhome hardening.
- **netcage argv composition** — anonbox never touches podman; it composes netcage argv (`run`/`start`/`verify`/`forward`/`ports`, `--allow-direct`, `-v home`, and, when available, netcage's `/etc`-identity machine flags), exactly as anon-pi drives netcage today. netcage's jail-intrinsic host-recon binds (`/etc/passwd`, `/etc/machine-id`) are being added on the netcage side (ADR-0013 extension); anonbox composes them when the netcage on PATH supports them, so it must feature-detect the netcage version/capability.
- **anoncore (published, v0.1.0)** — the shared, substrate-agnostic account/seed/marker/elevation core for the anon* family (a library, no binary; `github.com/wighawag/anoncore`). anonbox imports its account-hardening half (`provision` with no sudo/wheel grant, `seedhome`, `ui`, `sudoprobe`, `marker`) but NONE of anonctl's egress machinery (nftables/shim/lanexempt), because netcage owns egress. Its README already names anonbox as a planned consumer.
- **netcage** — sibling (published): forces a container's TCP+DNS egress through a socks5h proxy by network namespace, fail-closed; ships a `verify` leak-test; a drop-in podman-shaped CLI. The jail anonbox drives.
- **anonctl** — sibling (published): per-UID kernel-forced anonymized egress, fail-closed, ships a `verify` leak-test, already has an `exec` verb. The verb API anonbox mirrors.
- **anonseed** — sibling (being built): a Go config-seeder that writes a tool's config into a machine/account and declares its allow-direct exception (pi first). Composes with anonbox.
- **honest residual** — carried into the README in the netcage/anonctl house style: the shared-kernel/hardware/timing fingerprint (a machine is not a VM), and that the `anonbox-<name>` account boundary is plain Unix DAC (on-host unlinkability/discoverability, NOT hard containment against host root).

## Conventions

Standing per-change rules agents must follow in this repo.

<!-- No standing per-change rule set yet (no changeset / CHANGELOG / news-fragment convention). Add yours here, or delete this section. For enforcement, wire your own check into the `dorfl.json` `verify` gate. -->

## Skills this repo uses

- Required: `setup` (onboarding/migration), `to-spec`, `to-task`.
- Recommended: `review`, `grill-me`.
