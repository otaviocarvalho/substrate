# ADR-0001: Barceloneta KVM pilot — single-node k3s now, microVM next

**Status:** Proposto (amended 01/10/2026: placement decided — bare host)
**Date:** 2026-10-01
**Deciders:** Otavio Carvalho, Hermes (agent)
**Scope:** this fork's deployment target (barceloneta NUC pilot) and the exploration
roadmap on top of it. Upstream code changes land here as exploration branches, never
as silent divergence from `agent-substrate/substrate`.

## Context

- This fork exists to bend Substrate to a pilot we cannot run upstream-as-is:
  a **single-node k3s** on a home NUC first, then the **microVM path** once usable.
- Pilot host (verified by direct inspection 01/10/2026): barceloneta NUC — bare metal
  (`systemd-detect-virt` → `none`), **real `/dev/kvm`** (VT-x on, Core 3 100U, 6C/8T,
  15 W), 16 GB RAM (~12 GB available), 43 GB free NVMe, Proxmox host with LXC guests
  ct100–102, and a **production bot (maquinista) already living there**.
- **Placement amendment (01/10/2026, Otavio):** the box is **treated as dedicated**.
  ct100–102 (pihole / navidrome / jellyfin) are playgrounds, **not actively used** —
  breakage during the pilot is explicitly accepted. The **only protected residents**
  are the maquinista bot and Tailscale ssh access (our only path to the box).
- Upstream ground truth (verified in-repo 21/09 + 01/10/2026):
  - Workload isolation today = **gVisor (`runsc`) on pods** (AGENTS.md §security);
    gVisor is a userspace kernel (syscall interception) — **no `/dev/kvm` needed**,
    so the NUC runs it like any x86_64 Linux box.
  - K8s-native: actors on pods, deploy via `manifests/ate-install/`
    (incl. `sandboxconfig-gvisor.yaml`); supports "latest stable Kubernetes + previous
    minor" (README); dev/e2e flow assumes kind/GCP tooling — **single-node k3s is
    untested upstream**.
  - `cmd/ateom-gvisor` runs `runsc` checkpoint/restore inside worker pods;
    `cmd/ateom-microvm` + `manifests/microvm/` exist (checked 01/10) — the microVM
    path is materializing upstream; x86_64-centric throughout.
- Downstream consumer: maquinista ADR-0002 (executor port) / 0003 (Substrate adapter,
  tenant model) / 0004 (NUC as pilot host). This roadmap produces the evidence for
  maquinista's G-00a–d gates; maquinista keeps tmux as default executor regardless.

## Explorations (the roadmap)

Dependency order; each with hypothesis and success criterion.

- [x] **E-00 — k3s single-node bring-up + runsc RuntimeClass** (~0.3 d, directly on
  the host)
  Hypothesis: `runsc` works as a containerd RuntimeClass handler on k3s on this NUC.
  Success: an actor pod runs under `runsc`; `sandboxconfig-gvisor.yaml` applies clean.
  Budget (feeds G-00a): pilot stack capped ~8 GB RAM / 30 GB disk; the **bot and
  Tailscale ssh stay healthy** before/after (measured) — ct100–102 are expendable.

  **Result (2026-10-01, 21:57–22:12 CEST): GO.** k3s v1.36.5+k3s1 (stable channel),
  installed with `--data-dir /home/k3s --write-kubeconfig-mode 644 --disable
  traefik --disable servicelb --disable metrics-server` (coredns + local-path
  kept). gVisor `release-20260928.0` (SHA256 verified). Three gotchas on record
  for the next box:
  (1) the old `storage.googleapis.com/gvisor/releases` URLs 404 now — releases
  moved to GitHub, and the tarball ships a REQUIRED `gvisor-bin/` sidecar dir
  (`gvisor_sentry`, `runsc-fd-parking`, `runsc-metric-server`) next to `runsc` +
  `containerd-shim-runsc-v1`; default `--sidecar-usage-policy=STRICT` aborts every
  sandbox without it (`stat /usr/local/bin/gvisor-bin/gvisor_sentry: no such file`);
  (2) k3s's containerd 2.x only honors custom runtimes under the NEW plugin
  namespace — template block must be
  `[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runsc]` with
  `runtime_type = 'io.containerd.runsc.v1'`; the legacy `io.containerd.grpc.v1.cri`
  name renders into the file but is silently ignored (cost one restart cycle);
  (3) the template lives at `<data-dir>/agent/etc/containerd/config.toml.tmpl`
  (= `/home/k3s/agent/etc/containerd/config.toml.tmpl` here), built by copying the
  generated `config.toml` and appending the block.
  Proof: `runsc-test` pod (busybox, `runtimeClassName: gvisor`, RuntimeClass
  handler `runsc`) went Ready in 4 s; in-pod `dmesg` prints the gVisor boot banner
  on kernel 7.0.14-19-pve. Budget vs caps: available RAM 12.70 GB → 12.31 GB after
  k3s idle + one runsc sandbox (stack cost ≈ 0.4 GB of the 8 GB cap);
  `/home/k3s` = 253 MB of the 30 GB cap; load 0.15. Protected set intact after
  install + restart: maquinista active, Tailscale ssh up, ct100–102 running.
- [ ] **E-01 — manifest delta audit: upstream → k3s** (0.5–1 d) — *do first, falsifies
  the pilot's load-bearing assumption*
  Apply `manifests/ate-install` verbatim on k3s; record every failure and the minimal
  fix. Fixes land here as `k3s-delta/*` branches. Success: documented patch set + green
  apply; a written verdict on whether upstream would take the changes.
- [ ] **E-02 — PTY fidelity probe** (0.5–1 d; feeds maquinista G-00b)
  Hypothesis: an interactive agent TUI inside an actor yields a stream clean enough
  for MonitorProfile-style transcript scraping. Success: no torn reads, echo semantics
  known, written go/no-go for relay-through-Substrate.
- [ ] **E-03 — transcript egress via ate-env** (0.5–1 d; feeds G-00c)
  Ship JSONL transcript lines out of sandboxes using per-actor fs ops (ate-env is
  alpha). Measure per-append latency + auth overhead; stream vs poll. Success: measured
  cost table → per-runner go/no-go.
- [ ] **E-04 — microVM probe** (~1 d; gated on upstream) — `/dev/kvm` is direct on the
  host, no passthrough needed
  `cmd/ateom-microvm` + `manifests/microvm/` on the NUC; target boot + snapshot
  < 5 s. Trigger: E-01 lands AND upstream microvm manifests confirmed usable
  (directory exists as of 01/10 — audit its maturity before starting).
- [ ] **E-05 — guardrails for the protected set** (~0.5 d)
  Non-sacrificial set = **maquinista bot + Tailscale ssh/deploy path** only. Failure
  drills MAY take down ct100–102 (accepted); post-drill chores: confirm pihole/navidrome/
  jellyfin recover, `k3s-uninstall` reverses iptables/cgroup state, bot unit restarts
  clean. Success: a drill that breaks everything else and leaves the bot answering.

## Options considered (where the pilot stack lives on the NUC)

| Option | Verdict |
|---|---|
| **Bare NUC host (k3s directly on the existing Proxmox VE/Debian host)** | **DECIDED (Otavio, 01/10)** — box treated as dedicated; ct100–102 sacrificial. ~0.3 d; real `/dev/kvm` direct for E-04; no NAT hop; full ~12 GB to the pilot |
| Proxmox VM (KVM inside) | Rejected for now — extra layer on a box treated as dedicated (+0.2 d, ~1 GB RAM, NAT hop). Revisit trigger if host-level damage reaches the bot |
| LXC guest | Rejected — nested k8s + runsc inside LXC is fragile (cgroups, AppArmor, kvm passthrough) |
| Hetzner Cloud VP | Fallback only — gVisor works today, ~€26/mo, microVM never (no nested virt — vendor FAQ, fetched 21/09/2026) |

## Decision

Explore in the order above, **directly on the host** (k3s on the existing Proxmox VE /
Debian install). The trade we accept: k3s colonizes host networking (iptables/CNI,
sysctls, kernel modules) and the pilot shares the kernel with the bot — mitigated by
the E-05 protected set (bot + Tailscale) and the RAM cap, with ct100–102 as the
explicitly accepted blast radius. tmux stays maquinista's default executor — this fork
only needs to prove isolation + PTY + egress. Sync discipline: rebase exploration
branches on upstream `main` before each exploration; the only intended long-lived
branches are `k3s-delta/*`.

## Effort estimate

E-00 0.3 + E-01 0.5–1 + E-02 0.5–1 + E-03 0.5–1 + E-05 0.5 ≈ **2.3–4.3 dev days**
to a pilot go/no-go (E-04 extra, upstream-gated).
Load-bearing assumption to falsify first: **E-01** — that Substrate's manifests run on
single-node k3s without hidden GCP/full-cluster assumptions.

## Revisit triggers

- Upstream breaking changes (project self-declares pre-stability) → re-run the delta audit.
- `manifests/microvm/` matures → pull E-04 forward.
- **Host-level damage reaches the protected set** (bot down, Tailscale/deploy path
  broken by CNI/cgroup state) → fall back to VM placement, or Hetzner VP.
- Scale-out trigger fires (pilot validated, > ~6 sandboxes needed) → dedicated metal,
  where bare-host k3s is the natural default anyway.

## References

- `agent-substrate/substrate` — inspected 21/09 + 01/10/2026 (this fork's upstream)
- `agent-substrate/env` (ate-env) — cloned 01/10/2026; alpha API
- `google/ax` — CRDs (Task/Workspace/Gateway/Model) inspected 21/09/2026
- maquinista ADR-0002/0003/0004 + `references/substrate-ax-integration.md` (fact base)
- Hetzner FAQ nested-virt quote — fetched 21/09/2026
