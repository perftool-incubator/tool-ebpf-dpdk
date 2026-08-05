# tool-ebpf-dpdk

eBPF/perf-based CPU profiling and flamegraph generation tool for the [perftool-incubator](https://github.com/perftool-incubator) / [crucible](https://github.com/perftool-incubator/crucible) benchmarking ecosystem.

## Status

**Phase 1: complete** -- perf-based PMD profiling with flamegraph generation.

| Milestone | Status |
|-----------|--------|
| PMD thread auto-discovery (OVS-DPDK + generic DPDK) | Complete |
| Targeted `perf record -t <tid>` on PMD threads | Complete |
| FlameGraph SVG generation (interactive, browser-viewable) | Complete |
| Speedscope-compatible collapsed stacks (.folded) | Complete |
| Traffic-aware filtering (idle period exclusion) | Complete |
| CDM metrics (top function CPU%, sample counts) | Complete |
| Crucible/rickshaw integration (rickshaw.json, workshop.json) | Complete |
| Built-in stack collapser fallback | Complete |
| bpftrace scheduling interference probes | Phase 2 |
| Trial-aware flamegraph splitting (binary search) | Phase 3 |
| Differential flamegraphs (A/B comparison) | Phase 4 |

## Overview

tool-ebpf-dpdk answers the question **"where are DPDK PMD thread CPU cycles going?"** by producing CPU flamegraphs targeted at OVS-DPDK and testpmd Poll Mode Driver threads during crucible benchmark runs.

It fills the diagnostic gap between:
- **tool-dpdk** (telemetry counters -- *what* happened) and
- **tool-ovs** (PMD busy/idle stats -- *how much* is happening)

by showing *where* in the code the cycles are spent:

```
flamegraph reveals:
  51%  dpcls_lookup          ← megaflow classifier (EMC cache thrashing)
  28%  miniflow_extract      ← packet parsing overhead
  11%  conntrack_execute     ← conntrack bottleneck (if enabled)
   5%  netdev_send           ← vhost-user TX
   3%  dp_netdev_upcall      ← flow miss slow path
```

### How It Complements Other Tools

| Question | Tool | Answer |
|----------|------|--------|
| How many packets flowed? | tool-dpdk | 14.2 Mpps, 0 drops |
| How busy are PMD threads? | tool-ovs | 78% busy, 1200 flow misses/sec |
| **Where are PMD cycles spent?** | **tool-ebpf-dpdk** | **51% in dpcls_lookup (flamegraph)** |
| **What preempts PMD threads?** | **tool-ebpf-dpdk** | **ksoftirqd on core 4 (Phase 2)** |

## Usage with Crucible

Add `tool-ebpf-dpdk` to the `tool-params` section of your run file.

### Single Instance (basic)

```json
"tool-params": [
    { "tool": "sysstat" },
    { "tool": "procstat" },
    { "tool": "ebpf-dpdk", "params": [
        { "arg": "target", "val": "ovs-vswitchd" },
        { "arg": "frequency", "val": "99" }
    ]}
]
```

### Multi-Instance (recommended for OVS + testpmd topologies)

When profiling both OVS on the compute host and testpmd on a remote server, use separate tool instances with `opt-in` deployment to target each process on its correct host:

```json
"tool-params": [
    { "tool": "sysstat" },
    { "tool": "procstat" },
    {
        "tool": "ebpf-dpdk",
        "id": "ebpf-dpdk-ovs",
        "deployment": "opt-in",
        "opt-tag": "has-ovs",
        "params": [
            { "arg": "target", "val": "ovs-vswitchd" },
            { "arg": "frequency", "val": "99" }
        ]
    },
    {
        "tool": "ebpf-dpdk",
        "id": "ebpf-dpdk-testpmd",
        "deployment": "opt-in",
        "opt-tag": "has-testpmd",
        "params": [
            { "arg": "target", "val": "testpmd" },
            { "arg": "frequency", "val": "99" },
            { "arg": "background-discovery", "val": "yes" }
        ]
    }
]
```

Then in your endpoint host definitions, tag each host with the appropriate `tool-opt-in-tags`:

```json
"endpoints": [{
    "type": "remotehosts",
    "config": [
        {
            "host": "compute-host.example.com",
            "tool-opt-in-tags": "has-ovs"
        },
        {
            "host": "server-host.example.com",
            "tool-opt-in-tags": "has-testpmd"
        }
    ]
}]
```

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `target` | `auto` | Which DPDK processes to profile: `ovs-vswitchd`, `testpmd`, `all`, `auto` |
| `frequency` | `99` | perf sampling frequency in Hz (99 avoids timer aliasing) |
| `call-graph` | `dwarf` | Call graph unwinding mode: `dwarf`, `fp`, `lbr` |
| `perf-extra-opts` | (empty) | Additional options passed to `perf record` |
| `retry-interval` | `10` | Seconds between PMD discovery retries |
| `retry-max` | `18` | Max discovery retry attempts |
| `max-total-wait` | `180` | Max seconds to wait for PMD thread discovery |
| `background-discovery` | `no` | Enable background daemon mode for late-starting processes (see below) |

### Background Discovery Mode

When `background-discovery=yes`, the start script exits immediately (unblocking the roadblock) and spawns a background daemon that continuously polls for PMD threads. Once threads are found, the daemon launches `perf record`. This is essential for **testpmd** which starts after the benchmark client begins sending traffic.

Use background discovery when:
- The target process starts **after** the tool start phase (e.g., testpmd)
- You see `no-pmd-threads-found` errors with the default synchronous mode

Do **not** use background discovery for `ovs-vswitchd` — OVS is always running before the benchmark starts, so synchronous discovery works reliably.

### Minimal Example

Profile OVS-DPDK with defaults (auto-discovers PMD threads):

```json
{ "tool": "ebpf-dpdk" }
```

## Output

After a run, tool-ebpf-dpdk produces these files in the tool data directory:

```
tool-data/profiler/remotehosts-1-ebpf-dpdk-ovs-3/ebpf-dpdk-ovs/
├── perf-script.txt.gz                        Compressed perf script text output
├── flamegraph-ovs-vswitchd-full.svg          Full-run flamegraph (interactive SVG)
├── flamegraph-ovs-vswitchd-active.svg        Traffic-only flamegraph (idle filtered)
├── flamegraph-ovs-vswitchd.folded.xz         Collapsed stacks (speedscope-compatible)
├── pmd-discovery.json                        PMD thread metadata
├── pmd-discovery-stderr.txt                  Discovery log (shows method used)
├── ebpf-dpdk-begin-ms.txt                    Profiling start timestamp (epoch ms)
├── ebpf-dpdk-end-ms.txt                      Profiling end timestamp (epoch ms)
├── metric-data-0.json.xz                     CDM metric descriptors
├── metric-data-0.csv.xz                      CDM metric samples
└── post-process-data.json                    Rickshaw manifest
```

## Viewing Flamegraphs

### Option A: Open SVG in browser (zero setup)

```bash
firefox /var/lib/crucible/run/latest/run/tool-data/profiler/remotehosts-1-ebpf-dpdk-1/ebpf-dpdk/flamegraph-ovs-vswitchd-full.svg
```

The SVG is interactive -- hover for function names and sample counts, click to zoom.

### Option B: Speedscope (interactive web viewer)

```bash
xzcat .../flamegraph-ovs-vswitchd.folded.xz > /tmp/profile.folded
```

Drag the file to [speedscope.app](https://www.speedscope.app) (runs in-browser, nothing uploaded). Provides three views:
- **Time Order** -- chronological stack timeline
- **Left Heavy** -- traditional flamegraph (largest frames sorted left)
- **Sandwich** -- select any function to see all callers and callees

### Option C: KDAB Hotspot (desktop GUI)

```bash
xzcat .../perf.data.xz > /tmp/perf.data
hotspot /tmp/perf.data
```

Full GUI with per-thread timeline, flamegraph, and top-down/bottom-up views.

### Option D: Netflix FlameScope (perturbation hunting)

```bash
xzcat .../perf.data.xz > /tmp/perf.data
perf script -i /tmp/perf.data --no-inline > /tmp/profile.perf
```

Load in [FlameScope](https://github.com/Netflix/flamescope) for subsecond-offset heatmaps that reveal periodic CPU spikes.

## PMD Thread Discovery

The tool uses a two-tier discovery mechanism in `pmd-discovery.py`:

### OVS-DPDK (two-tier)

1. **Primary**: `ovs-appctl dpif-netdev/pmd-rxq-show` — provides rich metadata (rxq mapping, numa node, core assignments). Requires `ovs-appctl` to be installed.
2. **Fallback**: `/proc/<pid>/task/*/comm` scan for `pmd-c*` threads — works inside containers without the `openvswitch` package. Derives NUMA node from sysfs topology.

The fallback is critical because the `ebpf-dpdk` container image does not include `openvswitch` (and shouldn't — it would add 50MB+ of unnecessary dependencies). Since the container runs with `--pid=host` access, `/proc` scanning works reliably.

### Generic DPDK (testpmd, l3fwd, etc.)

Scans `/proc/<pid>/task/*/comm` for threads named `lcore-worker-*` after finding the DPDK process via `pgrep -f <process_name>`.

### Discovery Retries

Discovery retries every 10 seconds (up to `max-total-wait`, default 180s). If no PMD threads are found:
- **Synchronous mode**: the tool logs a warning and exits gracefully
- **Background-discovery mode**: the daemon keeps polling until the stop signal is received

## Supported Scenarios

| Scenario | Mode | Support |
|----------|------|---------|
| OVS-DPDK + testpmd (STL, binary search) | `target=ovs-vswitchd` | Full |
| OVS-DPDK + testpmd (ASTF, binary search) | `target=ovs-vswitchd` | Full |
| OVS-DPDK + testpmd (ASTF, one-shot) | `target=ovs-vswitchd` | Full (best flamegraph quality) |
| Bare-metal testpmd (no OVS) | `target=testpmd` | Full |
| VM testpmd (via QEMU vCPU threads) | `target=testpmd` | Requires host PID namespace |
| Kubernetes OVS-DPDK | `target=ovs-vswitchd` | Requires hostPID |
| Custom DPDK app | `target=auto` | Auto-discovers lcore-worker threads |

## Architecture

```
Collection (profiler engine)         Post-Processing (controller)
────────────────────────────         ────────────────────────────
pmd-discovery.py                     ebpf-dpdk-post-process
  ├─ ovs-appctl (if available)         ├─ decompress perf-script.txt.gz
  └─ /proc scan (fallback)             ├─ stackcollapse-perf.pl (or builtin)
       │                                │     ├─→ flamegraph.pl → SVG
       ▼                                │     └─→ .folded.xz (speedscope)
perf record -F 99 -g -t TIDs          ├─ traffic window detection
  (runs entire iteration)               │     └─→ active.svg
       │                                ├─ top function extraction
       ▼                                │     └─→ CDM metrics (per-instance source)
ebpf-dpdk-stop                         └─ post-process-data.json
  ├─ kill -SIGINT perf
  ├─ perf script → perf-script.txt
  └─ gzip perf-script.txt
```

Key design decisions:
- `perf script` runs during the stop phase (engine container has `perf`) rather than during post-processing (controller does not have `perf`)
- Compression uses `gzip` for speed (120s time budget for stop phase)
- Each tool instance emits CDM metrics under its own source name (e.g., `ebpf-dpdk-ovs`, `ebpf-dpdk-testpmd`)

## Dependencies

### Runtime (engine container image — defined in workshop.json)

| Package | Source | Purpose |
|---------|--------|---------|
| `perf` | Distro | CPU profiling and `perf script` during stop |
| `python3` | Distro | PMD discovery |
| `xz` | Distro | Compression utilities |
| FlameGraph toolkit | [github.com/brendangregg/FlameGraph](https://github.com/brendangregg/FlameGraph) | SVG generation (optional, builtin fallback exists) |

### Post-processing (controller container)

| Dependency | Source | Purpose |
|------------|--------|---------|
| `toolbox.metrics` | `subprojects/core/toolbox` | CDM metric emission |
| `gzip` | Base image | Decompression of `perf-script.txt.gz` |

Note: `perf` is **not** required in the controller. The `perf script` conversion happens during the stop phase inside the engine container where `perf` is installed. The controller only reads the pre-generated text output.

## Example Results

From a trafficgen benchmark run with OVS-DPDK (compute host) and testpmd (server host):

### CDM Metric Query

```bash
# Query OVS PMD top functions
$ crucible get metric --run <run-id> --source ebpf-dpdk-ovs --type top1-function-pct --breakout function

                                                        source             type                function   value
 ebpf-dpdk-ovs top1-function-pct rte_vhost_dequeue_burst      19.92

# Query all top 5 functions
$ crucible get metric --run <run-id> --source ebpf-dpdk-ovs --type top2-function-pct --breakout function
$ crucible get metric --run <run-id> --source ebpf-dpdk-ovs --type top3-function-pct --breakout function
...
```

### Sample Output (OVS-DPDK, 10 PMD threads, 60s run)

```
source: ebpf-dpdk-ovs
  perf-samples:      25,231
  top1-function-pct: rte_vhost_dequeue_burst    19.92%
  top2-function-pct: dp_netdev_process_rxq_port 15.74%
  top3-function-pct: rxq_cq_process_v           13.44%
  top4-function-pct: [unknown]                   6.42%
  top5-function-pct: pmd_thread_main             5.17%

source: ebpf-dpdk-testpmd
  perf-samples:      2
  top1-function-pct: exit_to_user_mode_loop     50.00%
  top2-function-pct: _raw_spin_unlock_irq       50.00%
```

### PMD Discovery Log

```
ebpf-dpdk: discovered 10 OVS PMD thread(s) via /proc
  core 10: TID 2066 [pmd-c10/id:11]
  core 11: TID 2071 [pmd-c11/id:16]
  core 12: TID 2067 [pmd-c12/id:12]
  core 13: TID 2064 [pmd-c13/id:9]
  core 14: TID 2072 [pmd-c14/id:17]
  core 15: TID 2073 [pmd-c15/id:18]
  core 16: TID 2069 [pmd-c16/id:14]
  core 17: TID 2065 [pmd-c17/id:10]
  core 18: TID 2070 [pmd-c18/id:15]
  core 19: TID 2068 [pmd-c19/id:13]
```

### Available Metric Types

| Source | Type | Class | Description |
|--------|------|-------|-------------|
| `ebpf-dpdk-<id>` | `top-function-pct` | percentage | Highest CPU% function |
| `ebpf-dpdk-<id>` | `top1-function-pct` ... `top5-function-pct` | percentage | Top 5 functions by CPU% |
| `ebpf-dpdk-<id>` | `perf-samples` | count | Total perf samples collected |
| `ebpf-dpdk-<id>` | `perf-samples-active` | count | Samples during active traffic window |

The `<id>` suffix matches the tool instance ID from the run file (e.g., `ovs`, `testpmd`).

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `no-pmd-threads-found` | Target process not running at tool start time | Use `background-discovery=yes` for late-starting processes |
| Roadblock timeout at `start-tools-end` | PMD discovery taking too long | Reduce `max-total-wait` or switch to `background-discovery` |
| Stuck at `stop-tools-end` | Large perf.data + slow compression | Fixed: stop script has 120s time budget with gzip |
| Zero-value CDM metrics | Missing epoch timestamps in metric data | Fixed: begin/end timestamps recorded during collection |
| `perf: command not found` during post-process | perf not in controller image | Fixed: `perf script` now runs during stop phase (engine has perf) |
| OVS PMDs not discovered in container | `ovs-appctl` not installed in engine image | Fixed: `/proc` fallback doesn't require OVS packages |

## Documentation

- [Technical Architecture Document](docs/tool-ebpf-dpdk-technical-architecture.md) -- end-to-end design, topology mapping, phased delivery plan, future enhancements

## Roadmap

- **Phase 2**: bpftrace scheduling interference probes (`bpf/pmd_sched.bt`). Off-CPU flamegraphs for PMD preemption analysis.
- **Phase 3**: Trial-aware flamegraph splitting. Generate per-trial flamegraphs during binary search using tool-dpdk telemetry timestamp correlation.
- **Phase 4**: Differential flamegraphs for A/B run comparison. Automated regression detection.

## License

Apache License 2.0 -- see [LICENSE](LICENSE).
