# ADR-0001: Barceloneta KVM pilot — single-node k3s now, microVM next

**Status:** Proposto
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
  ct100–102, and a **production bot (maquinista) already living there** — nothing in
  this roadmap may break it (coexistence gate).
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

- [ ] **E-00 — k3s single-node bring-up + runsc RuntimeClass** (~0.5 d)
  Hypothesis: `runsc` works as a containerd RuntimeClass handler on k3s on this NUC.
  Success: an actor pod runs under `runsc`; `sandboxconfig-gvisor.yaml` applies clean.
  Budget proposal (feeds G-00a): pilot stack capped ~8 GB RAM / 30 GB disk; bot +
  ct100–102 measured unaffected before/after.
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
- [ ] **E-04 — microVM probe** (~1 d; gated on upstream) — needs the NUC's real KVM
  `cmd/ateom-microvm` + `manifests/microvm/` on the NUC; target boot + snapshot
  < 5 s. Trigger: E-01 lands AND upstream microvm manifests confirmed usable
  (directory exists as of 01/10 — audit its maturity before starting).
- [ ] **E-05 — coexistence + rollback guardrails** (~0.5 d)
  Resource caps, failure drills, documented teardown (VM delete / `k3s-uninstall`)
  leaving the bot and ct100–102 untouched.

## Options considered (where the pilot stack lives on the NUC)

| Option | Verdict |
|---|---|
| Proxmox VM (KVM inside) | **Recommended** — clean isolation from the prod bot, real `/dev/kvm` inside for E-04, snapshot rollback. ~0.5 d |
| Bare NUC host (k3s directly) | Simplest networking, but experiments share the kernel with the production bot. ~0.3 d, higher blast radius |
| LXC guest | Lightest, but nested k8s + runsc inside LXC is fragile (cgroups, AppArmor, kvm passthrough). Rejected |
| Hetzner Cloud VP | gVisor works today, ~€26/mo, microVM never (no nested virt — vendor FAQ, fetched 21/09/2026). Fallback only if E-00/E-05 coexistence fails |

## Decision

Explore in the order above, inside a **Proxmox VM** on the NUC. tmux stays maquinista's
default executor — this fork only needs to prove isolation + PTY + egress. Sync
discipline: rebase exploration branches on upstream `main` before each exploration;
the only intended long-lived branches are `k3s-delta/*`.

## Effort estimate

E-00 0.5 + E-01 0.5–1 + E-02 0.5–1 + E-03 0.5–1 + E-05 0.5 ≈ **2.5–4.5 dev days**
to a pilot go/no-go (E-04 extra, upstream-gated).
Load-bearing assumption to falsify first: **E-01** — that Substrate's manifests run on
single-node k3s without hidden GCP/full-cluster assumptions.

## Revisit triggers

- Upstream breaking changes (project self-declares pre-stability) → re-run the delta audit.
- `manifests/microvm/` matures → pull E-04 forward.
- E-00/E-05 coexistence failure → fall back to Hetzner VP (gVisor-only), or demote the
  NUC to maquinista's scale-out trigger as in ADR-0004.

## References

- `agent-substrate/substrate` — inspected 21/09 + 01/10/2026 (this fork's upstream)
- `agent-substrate/env` (ate-env) — cloned 01/10/2026; alpha API
- `google/ax` — CRDs (Task/Workspace/Gateway/Model) inspected 21/09/2026
- maquinista ADR-0002/0003/0004 + `references/substrate-ax-integration.md` (fact base)
- Hetzner FAQ nested-virt quote — fetched 21/09/2026
