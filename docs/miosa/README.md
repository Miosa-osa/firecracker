# MIOSA Firecracker fork

This is MIOSA's fork of the upstream Firecracker VMM. It exists for three reasons:

1. **Version identity** — production runs a specific VMM build; the fork records exactly which upstream commit that is, so a certified sandbox generation can name its VMM the way it names its kernel and rootfs.
2. **Reproducible builds** — we build our own `firecracker` binary from a pinned source tree instead of trusting a downloaded artifact of unknown provenance.
3. **A home for patches with evidence** — when a measured problem or a latency frontier requires a VMM change, it lands here on a version branch, canaried, never hot-swapped.

## Branches

| Branch | Upstream base | Meaning |
|---|---|---|
| `main` | tracks upstream | untouched mirror |
| `miosa-v1.12` | `v1.12.0` (`15e74906`) | **what production runs today**, patch-zero |
| `miosa-v1.16` | `v1.16.1` (`2038188f`) | **upgrade target**, patch-zero |

## Why upgrade v1.12.0 → v1.16.1

Production runs stock v1.12.0. Upstream is at v1.16.1 — four minor versions of fixes we do not have. The upgrade is worth doing on its own merits, and it resolves things we have actually measured or that specifically affect our hardware:

| We observed / it affects us | Upstream fix now in v1.16.1 |
|---|---|
| Snapshot-resumed sandboxes wake with a clock days in the past | `fix(kvm-clock): do not jump monotonic clock on restore` (Apr 2026), plus the earlier `KVM_KVMCLOCK_CTRL before resume` work |
| Fleet is **all-AMD EPYC**, running a VMM without recent AMD host-CPU feature/MSR corrections | AMD host CPU-feature checks + `MSR_TSC_RATE` handling |
| Block-device fairness / VIRTIO notification correctness under load | block "notify guest once" + VIRTIO notification compliance fixes |

The guest still needs its own clock re-sync step on resume regardless of the VMM fix — belt and suspenders.

## Upgrade is a generation-ledger event, not a hot swap

The VMM version is part of a sandbox's certified generation (a restored snapshot expects the VMM it was baked under). So this upgrade runs the same path as any artifact change:

1. Build `firecracker` from `miosa-v1.16`, record its sha256 + version in the generation manifest.
2. Rebuild the guest kernel (`miosa-kernels`, 6.1.158) against the new VMM if needed; pair them in one generation.
3. Bake one new snapshot generation on that (kernel, rootfs, VMM) triple.
4. Canary one host: two clean restores, cache-drop hash check, **clock-skew within SLO**, DNS/TLS after eviction, disk write+fsync.
5. Promote by generation with rollback armed; never overwrite the running binary in place.

## Known landmine before we add our own patches

If we later enable Firecracker's async `io_engine` (io_uring) or a non-default drive `cache_type` — both recommended by the platform architecture study for clone durability — the stock **seccomp filter will block the new syscalls**. The allowlist in `resources/seccomp/*.json` must be patched here first and canaried. This is a known failure mode, not a surprise to discover in production.

## What we deliberately did NOT do

- No patches on either version branch yet — production's issues were platform/artifact-contract correctness, not the VMM. Patch-zero until evidence says otherwise.
- Incremental/dirty-memory snapshot capture (the sub-second-resume frontier) is a future project on this fork, taken up only after the artifact-contract canary program certifies correctness and resume latency becomes the priority.
