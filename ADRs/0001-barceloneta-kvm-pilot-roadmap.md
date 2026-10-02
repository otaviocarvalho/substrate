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
- [x] **E-01 — manifest delta audit: upstream → k3s** (0.5–1 d) — *do first, falsifies
  the pilot's load-bearing assumption*
  Apply `manifests/ate-install` verbatim on k3s; record every failure and the minimal
  fix. Fixes land here as `k3s-delta/*` branches. Success: documented patch set + green
  apply; a written verdict on whether upstream would take the changes.

  **Progress (2026-10-02): demo sandbox suite green on k3s.** `ate-setup deploy demo
  sandbox` (fork branch `barceloneta-k3s`) builds with ko and pushes to the local
  registry `100.74.121.4:30500` (rustfs S3); atelet registry auth off; envoy cargo env
  exported; GKE PodMonitoring dropped. Creates atespace `ate-demo-sandbox`: 2×
  `sandbox-workerpool` pods Running (runsc), template `sandbox-template`
  (SANDBOX_CLASS_GVISOR) with golden snapshot published. Fork delta so far is deploy
  plumbing only — no Go code changes. Still open: apply the FULL `manifests/ate-install`
  set + written upstream verdict (this audit remains the falsification gate for E-04).

  **Re-run + closure (2026-10-02, upstream tip `6a35150e` → fork `0687c6ee`): E-01 CLOSED.**
  Trigger: the same-day upstream tracking merge changed 9 manifest files across 15
  commits, so the audit was re-run per its own revisit rule. Verdict on the merge delta:
  egress dataplanes consolidated (mitm folded into `atenet-egress.yaml`, +825; the
  1396-line `-with-sdsmint` variant and the `agentgateway-egress-mitm` component
  deleted upstream — neither was in our deploy path); `cordon-control-plane`
  restructured but still flag-gated (unused); kind-overlay consistency fix (not our
  path); ClusterTrustBundle v1 support is controller-internal (no static manifest
  refs; controller + CRD upgrade together). Fork delta vs upstream tip is docs +
  manifests + Dockerfile only (no Go, no generated proto) → no buf regen needed.
  Three new fork fixes landed, each found by deploy-and-observe:
  1. `egress-mitm-ca-pool` Secret: the consolidated egress mounts it non-optionally
     (`ca-state` volume); without it the pod loops on FailedMount and the rollout
     stalls while the old RS keeps serving. Bootstrap once with
     `ate-setup create egress-mitm-ca-pool` (idempotent, keeps existing).
  2. Credential-provider mounts REMOVED from base `atelet.yaml` (commit `6d87ab35`):
     upstream #917 added hostPath mounts whose config path is
     `/etc/srv/kubernetes/cri_auth_config.yaml` with `type: FileOrCreate` —
     FileOrCreate cannot create the missing GKE parent dir, so on k3s the pod loops
     on FailedMount forever. Mirrors upstream's own kind-overlay remedy.
  3. `--gcp-auth-for-image-pulls=false` DROPPED (commit `0687c6ee`): #917 removed the
     flag; the binary rejected it (`unknown flag`) and crashlooped. With no kubelet
     credential providers configured, pulls are anonymous — correct for our
     auth-disabled node-local registry. mTLS flags verified still present.
  Process lesson: ate-setup applies base manifests via SERVER-SIDE apply —
  strategic-merge `$patch: delete` directives are rejected (`field not declared in
  schema`); fork owns the base file, so blocks are deleted outright.
  Validation: full redeploy green on `v0.3.0-46-g0687c6ee` (api-server 2/2, atelet
  1/1, atenet-egress 3/3 on the consolidated mitm config, router/controller/postgres/
  rustfs up), then counter demo re-run as the end-to-end check: POST → suspend → POST
  both counters continued (memory 2, file 2) — snapshot/restore intact on upstream tip.
  Ops incident same day: the box's Tailscale MagicDNS stub (100.100.100.100) began
  SERVFAILing all external names mid-session (worked ~30 min earlier; suspected
  pihole upstream, which Otavio says can be shut down if needed). Node unblocked via
  `tailscale set --accept-dns=false` + `nameserver 192.168.0.1` (router) in
  `/etc/resolv.conf` + CoreDNS restart — pod-level DNS verified. Tailscale
  networking itself untouched (it is the only way in). Side effect: .ts.net names no
  longer resolve from the box (ops references use TS IPs anyway).
  Remaining after closure: E-05 guardrails (only experiment left before pilot
  go/no-go); E-04 still upstream-gated (`manifests/microvm/`).
- [x] **E-02 — PTY fidelity probe** (0.5–1 d; feeds maquinista G-00b)
  Hypothesis: an interactive agent TUI inside an actor yields a stream clean enough
  for MonitorProfile-style transcript scraping. Success: no torn reads, echo semantics
  known, written go/no-go for relay-through-Substrate.

  **Result (2026-10-02): GO.** Probe: 1.9 MB static Go driver (stdlib-only PTY via
  `/dev/ptmx` ioctls, no module downloads) injected into a `sandbox-template` actor as
  44×60 KB base64 chunks through the atenet router `/process` endpoint (md5 verified
  end-to-end). Driver spawned `sh → busybox vi` on a real PTY (24×80 winsize), recorded
  every master read with `(timestamp, size)` metadata, and fed scripted keystrokes from
  files — all through plain HTTP POSTs, no actor-side tooling needed. Findings:
  (1) **PTY stream is byte-exact**: 13 recorded chunks sum exactly to the 972 B stream,
  no loss, no duplication; reassembled stream parses as 67/67 valid CSI sequences;
  vi drew status lines (`[Modified] 4/4 100%`), inserted text, and exited with a clean
  alt-screen teardown (`ESC[?1049l`) + `:wq` save (`5L, 97C`).
  (2) **Suspend/resume preserves the live process tree**: actor suspended (≈5 s, worker
  pod released) and resumed on a DIFFERENT worker pod with the SAME pids — driver and
  vi kept running; the held-open PTY master stayed writable across restore; post-resume
  keystrokes landed in the same vi buffer and `:wq` saved a file containing BOTH
  pre-suspend and post-resume text. This is checkpoint/restore of the sandbox, not a
  respawn-from-golden. Stream during the freeze window: exactly one 227.6 s gap, zero
  stale output dumped at restore.
  (3) **Echo semantics**: cooked phase (`sh`) echoes typed input as expected; vi runs
  the tty raw (typed chars not echoed — screen updates only). Transcript scrapers must
  parse the ANSI stream, not assume echo.
  (4) **Actors run under gVisor** (`/proc` exposes `gvisor/`, `sentry-meminfo`).
  (5) **Actor budget observed**: 1 GiB RAM, 2 CPU shares, uid 0, Alpine 3.24.1 +
  busybox only — no python3/script/tmux/socat; egress is default-deny (TLS to
  dl-cdn.alpinelinux.org reset ⇒ E-03 preview: transcript egress must go through
  atenet/ate-env fs ops, not direct network).
  (6) **`/process` is effectively at-least-once** (duplicate execution observed across
  retries) — arbitrary command execution must be idempotent or lock-protected; probe
  driver takes an `O_EXCL` lock at startup.
  Caveat: probe drove input+output through atenet `/process` (request/response); a
  live WebSocket/streaming relay (what maquinista G-00b would use) is NOT yet exercised
  — E-03 or a follow-up should stream the same stream before wiring MonitorProfile
  scraping. Artifacts: `~/e02-artifacts/` on the box (driver.go, pty.raw, pty.meta).
- [x] **E-03 — transcript egress via ate-env** — DONE 02/10/2026
  Plan: ship JSONL transcript lines out of sandboxes using per-actor fs ops (ate-env is
  alpha). Measure per-append latency + auth overhead; stream vs poll. Success: measured
  cost table → per-runner go/no-go. (Design note from E-02: direct egress is blocked
  inside actors — TLS reset by default-deny policy — so ate-env/atened is the only
  path; that constraint is now confirmed, not assumed.)

  Result (ate-env @ ab40c7b deployed on k3s — ns `ate-env`: api + 6 gVisor workers,
  template `default-template`; probe `~/code/env/e03probe/` on the box; workload =
  300-line × ~184 B JSONL transcript replayed at 20 lines/s ≈ 55 KB):

  **Verdict: GO** for per-runner transcript egress — live view via
  `StreamProcessOutputs` push (TUI-grade), durable copy via segmented-file pull at
  1 s polls. **NO-GO** for whole-file polling at scale, outside-initiated appends,
  and multi-tenant exposure as-is (no per-call auth).

  Cost table (client → port-forward → api → router → guest):
  - Per-op fixed cost ≈ 8–9 ms; shell round-trip 20.2 ms mean (n=20); ReadFile 4 KB
    8.8 ms / 256 KB 12.2 ms / 1 MB 40.5 ms (ceiling ≈ 26 MB/s). No per-call auth:
    client→api is plaintext gRPC carrying `x-env-id`/`x-env-atespace` metadata only —
    identity is network position; the api is the enforcement point G-00c must build.
  - P1 push (lines → process stdout → `StreamProcessOutputs{Follow:true}`): 300/300
    lines, one chunk per line (no coalescing), +2.17 s total over the 15 s nominal
    run (≈ 7 ms/line pipeline cost), max inter-chunk gap 107 ms.
  - P2 poll-whole-file (`ReadFile` every 1 s): staleness ≈ 2–3 s; wire 552 KB = 10×
    amplification with O(n) growth (a 10 MB transcript ⇒ ~10 MB/s waste; @250 ms
    polls → 1.88 MB = 34×).
  - P3 segmented pull (40-line segments, `test -f` pre-check, 1 s polls, tail
    segment re-read): staleness ≈ 2.3 s; wire 112 KB ≈ 2×; per-read 11–15 ms.

  Findings:
  (1) **WriteFile is O_TRUNC, no append RPC** — transcript appends must happen
  IN-guest (shell `>>`); ate-env writes serve bootstrap/config only. No LIST/GLOB
  RPC in the guest fs service → segment names must be predictable (poll N until
  miss).
  (2) **Push beats pull for liveness**: ~7 ms/line vs a 2–3 s poll floor; the floor
  is interval-dominated, not read-cost-dominated — faster polling buys little
  (250 ms polls: −0.8 s staleness for 3.4× wire cost).
  (3) **Alpha bug, to file upstream: warm-connection ReadFile on a missing path
  wedges** — no NotFound, no EOF; hangs until client timeout, while a fresh
  connection returns NotFound in 26 ms. Pollers must existence-check
  (`sh -c 'test -f'`, ~20 ms round-trip) and run per-read timeouts (probe: 15 s)
  to self-heal. One further suspected stream wedge after env idle/resume (first
  P3 run) — same class as E-02's suspend/resume caveat.
  (4) **Fork skew**: env@ab40c7b emits ActorTemplate fields this fork's proto
  rejects (`readyz` unknown; `snapshotsConfig` vs fork's `snapshotConfig`) — the
  manifest was patched by hand to deploy; the fork is behind upstream's
  ActorTemplate proto. Port the delta before E-04.
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

## Outcome log

**02/10/2026 — upstream-first contributions + wedge closure (WS-A/WS-B/WS-C):**

- **WS-B append-rpc**: `WriteFileRequest.append` (proto field 4, additive) is PR
  [agent-substrate/env#72](https://github.com/agent-substrate/env/pull/72) (open, ready for
  review): guest opens `O_CREATE|O_WRONLY|O_APPEND` on first stream message; 4 tests incl.
  e2e through real router+guest; verifier PASS against `.specs/features/append-rpc/checks.md`
  (C1–C5, coverage complete). Fork carries the cherry-pick + specs/tooling: env fork branch
  `barceloneta-k3s` @ 36c0274.
- **WS-A wedge**: not reproducible under controlled conditions — missing-path reads
  `NotFound` in ~25 ms (cold + warm), 150-op sequential load clean, forced suspend → warm
  read auto-resumed at 185 ms, composite follow-stream + poller + mid-run suspend 60/60
  clean. Filed upstream as
  [agent-substrate/env#73](https://github.com/agent-substrate/env/issues/73) with the
  bisector (`wedge`, `e03probe` — now in-repo under `.specs/tools/` on the env fork branch,
  superseding the scratch `~/code/env/e03probe/` copy).
- **LIST/GLOB**: deliberately cut from #72; proposed upstream as
  [agent-substrate/env#74](https://github.com/agent-substrate/env/issues/74)
  (single-directory glob + pagination; recursion/watch out of scope for v1).
- **WS-C upstream tracking**: `barceloneta-k3s` merged upstream/main @ 6a35150e (fork
  064000eb). Proto drift since merge-base 148df4b0 (`ateapi.proto` ±57, `atelet.proto` ±28)
  touches no fork code — fork delta is docs/manifests only. `atelet.yaml` merge conflict
  resolved keeping `--gcp-auth-for-image-pulls=false`: upstream's kubelet
  credential-provider mounts are GKE/EKS-specific and have no k3s equivalent here.

## References

- `agent-substrate/substrate` — inspected 21/09 + 01/10/2026 (this fork's upstream)
- `agent-substrate/env` (ate-env) — cloned on the box @ ab40c7b 02/10/2026; alpha API;
  deployed to k3s (ns `ate-env`); probe at `~/code/env/e03probe/`; probe + wedge bisector
  in-repo on env fork branch `barceloneta-k3s` at `.specs/tools/` (`otaviocarvalho/env`)
- `google/ax` — CRDs (Task/Workspace/Gateway/Model) inspected 21/09/2026
- maquinista ADR-0002/0003/0004 + `references/substrate-ax-integration.md` (fact base)
- Hetzner FAQ nested-virt quote — fetched 21/09/2026
