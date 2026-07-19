# MIOSA Firecracker production hardening & certification gates

Derived from upstream Firecracker production and snapshotting guidance (`docs/prod-host-setup.md`,
`docs/snapshotting/snapshot-support.md`, `docs/snapshotting/random-for-clones.md`) checked against
the live MIOSA fleet on 2026-07-19. These are the VMM/host-level gates a certified sandbox
generation must pass, alongside the platform artifact-contract gates.

## Fleet facts (measured 2026-07-19)

- Host CPU: **AMD Ryzen 9 9950X** (Zen 5) on sampled hosts. Host kernel 6.8. **cgroups V2 unified — good** (upstream flags cgroups V1 as a snapshot-latency killer; we are already on V2).
- Guest currently reports CPU brand **"AMD EPYC"** while the host is Ryzen 9950X — i.e. the guest already sees a CPUID that is not the host's, with **no explicit CPU template configured**. That divergence is unmanaged today.
- virtio-rng entropy device **is** attached pre-boot (`fast_boot.ex` → `Api.put_entropy_device`). Good, but insufficient alone (see entropy gate).

## Gate 1 — CPU template (currently MISSING, latent cross-host restore hazard)

Upstream: without a CPU template on x86_64, (a) `MSR_IA32_TSX_CTRL` is not preserved across snapshot restore, and (b) restoring a snapshot on a host whose CPU features differ from the bake host is unsafe — the guest may see features that vanished, causing `#UD`/crashes.

MIOSA implication: snapshots are baked on one host and restored on another (fast-boot slots are distributed fleet-wide). Today all sampled hosts are the same Ryzen 9950X, so it works by luck. The moment the fleet gains a different stepping or a genuine EPYC host (the procurement plan named EPYC 4545P), cross-host restore breaks with no warning.

**Gate:** define a MIOSA CPU template (a conservative common-denominator feature mask), apply it at boot before snapshot, and record `cpu_template_id` in the generation manifest. Placement must refuse to restore a generation on a host that cannot satisfy its template. Until the template exists, **placement must pin a generation's restores to the same CPU model it was baked on.**

## Gate 2 — Snapshot uniqueness / entropy (partial mitigation only)

Upstream: resuming a given snapshot more than once is **insecure** — every clone inherits the same kernel entropy pool, RNG seeds, and any secrets/identifiers generated pre-snapshot. MIOSA's fast-boot resumes ONE golden memory snapshot for MANY sandboxes — exactly the pattern the doc warns against.

Current mitigations: per-VM rootfs rebase + MMDS identity rotation (handled), and virtio-rng attached (feeds fresh entropy post-boot). Gap: nothing signals the guest kernel "you were restored, reseed now," so the entropy pool state is identical across clones until the guest happens to draw enough new randomness.

**Gate:** enable the **VMGenID** device (supported on our 6.1 guest kernel) so the guest detects restore and reseeds its CSPRNG; verify in the certification soak that two clones from one snapshot produce different `getrandom()` output immediately post-resume. Bake snapshots only *after* the guest kernel has finished booting (upstream: early-boot snapshots crash on resume).

## Gate 3 — Serial device unbounded output (host memory/storage risk)

Upstream: the guest can drive unbounded data through the 8250 serial device into the Firecracker process stdout, and can re-enable it even if disabled at boot. MIOSA must (a) boot guests with `8250.nr_uarts=0`, and (b) bound the Firecracker stdout sink (ring buffer / logrotate / named pipe), never an unbounded file. Ties to the platform "bounded command output" contract already in the artifact-contract slice.

## Gate 4 — Seccomp (keep default; patch deliberately)

Upstream: run the default (most-restrictive) seccomp filter; `--no-seccomp`/custom filters are not recommended for production. MIOSA runs stock v1.12.0 default filters — good. **When** we enable async `io_engine` (io_uring) or a non-default drive `cache_type` for clone durability (platform study recommendation), the default filter will block the new syscalls — the allowlist must be patched on this fork and canaried, not disabled.

## Gate 5 — Host patching cadence

Upstream: host kernel + microcode must track security advisories; guest kernel is owned in `miosa-kernels`. The VMM upgrade v1.12.0 → v1.16.1 (`miosa-v1.16`) is the current owed step and carries the kvm-clock restore fix (our measured resume clock-skew) plus recent AMD host-CPU/MSR corrections.

## Summary: what to add to the generation manifest

`cpu_template_id`, `vmgenid: enabled`, `vmm_version` + `vmm_sha256`, `kernel_version` + `kernel_sha256`, `seccomp: default|patched<reason>`. A generation is not certifiable until Gates 1–2 pass in the restore soak.
