# Tool-ebpf-dpdk

## Purpose
Crucible tool for eBPF/perf-based CPU profiling and flamegraph generation targeting OVS-DPDK and DPDK testpmd PMD threads. Produces interactive flamegraph SVGs, speedscope-compatible collapsed stacks, and CDM metrics for top function hotspots.

## Languages
- Bash: start/stop wrapper scripts (`ebpf-dpdk-start`, `ebpf-dpdk-stop`)
- Python: PMD discovery, post-processing, and CDM metric emission (`pmd-discovery.py`, `ebpf-dpdk-post-process`)

## Key Files
| File | Purpose |
|------|---------|
| `ebpf-dpdk-start` | Launches PMD discovery and perf recording with configured parameters |
| `ebpf-dpdk-stop` | Stops perf, runs `perf script`, and compresses output with gzip |
| `ebpf-dpdk-post-process` | Collapses stacks, detects traffic windows, generates flamegraphs, and emits CDM metrics |
| `pmd-discovery.py` | Discovers PMD thread TIDs via `ovs-appctl` or `/proc` scanning |
| `rickshaw.json` | Rickshaw integration: collector scripts, blacklist/whitelist |
| `workshop.json` | Engine image build requirements: perf, FlameGraph, python3 |
| `tool-metadata.json` | Machine-readable description and CDM-indexed status (consumed by `crucible tools list`) |
| `multiplex.json` | Parameter validation rules and `defaults` preset for multiplex (mirrors benchmark `multiplex.json`) |

## Configuration
- `--target <name>` — Target process for PMD discovery (`auto`, `ovs-vswitchd`, `dpdk-testpmd`, `testpmd`, `generic`; default: `auto`)
- `--frequency <hz>` — Perf sampling frequency in Hz (default: `99`)
- `--call-graph <mode>` — Call graph unwinding method (`dwarf`, `fp`, `lbr`, `no`; default: `dwarf`)
- `--perf-extra-opts <opts>` — Extra options passed directly to `perf record`
- `--retry-interval <seconds>` — Retry interval for PMD discovery in seconds (default: `10`)
- `--retry-max <count>` — Maximum PMD discovery retry attempts (default: `18`)
- `--max-total-wait <seconds>` — Maximum total wait time for PMD discovery in seconds (default: `180`)
- `--background-discovery <yes|no>` — Whether to run PMD discovery as a background daemon (default: `no`)

## Architecture

### Script Flow

1. **`ebpf-dpdk-start`** — Discovers PMD TIDs via `pmd-discovery.py`, launches `perf record` in background. Supports two modes:
   - **Synchronous** (default): discovers TIDs immediately, starts perf, blocks until perf exits
   - **Background discovery** (`--background-discovery yes`): exits immediately, spawns daemon that polls for TIDs

2. **`ebpf-dpdk-stop`** — Handles cleanup with a 120s time budget:
   - Writes stop-signal file (daemon checks this)
   - Sends SIGINT to perf process, waits for graceful exit
   - Runs `perf script --no-inline` to convert binary perf.data to text
   - Compresses perf-script.txt with gzip (removes raw perf.data)
   - Records end timestamp (`ebpf-dpdk-end-ms.txt`)

3. **`ebpf-dpdk-post-process`** — Python post-processor (runs in controller):
   - Reads pre-generated `perf-script.txt.gz` (no `perf` binary needed)
   - Collapses stacks via FlameGraph toolkit or built-in fallback
   - Detects traffic windows from sample density
   - Emits CDM metrics with proper epoch-ms timestamps
   - Uses tool instance name as metric source (e.g., `ebpf-dpdk-ovs`)

4. **`pmd-discovery.py`** — Reusable PMD thread discovery with two-tier OVS support:
   - **Primary**: `ovs-appctl dpif-netdev/pmd-rxq-show` (rich rxq metadata)
   - **Fallback**: `/proc/<pid>/task/*/comm` scan for `pmd-c*` threads (no OVS packages needed)
   - Generic DPDK: scans for `lcore-worker-*` threads

## Key Design Decisions

### perf script in stop phase (not post-process)
The controller container does NOT have `perf` installed. Running `perf script` during the stop phase (inside the engine container where perf exists) avoids this dependency mismatch and also reduces data transfer (text compresses better than binary perf.data).

### /proc-based OVS discovery fallback
The engine container's `workshop.json` only includes `perf`, `python3`, `xz`, and FlameGraph — no `openvswitch` package. The `/proc` fallback discovers OVS PMD threads by scanning `/proc/<pid>/task/*/comm` for the `pmd-cNN/` naming pattern, requiring only procfs access (available via `--pid=host`).

### Background discovery daemon
For processes that start after the tool (e.g., testpmd), the daemon polls continuously without blocking the roadblock. It writes status to `ebpf-dpdk-status.txt` and checks for the stop-signal file each iteration.

### Per-instance metric source naming
Each tool instance emits CDM metrics under its own source name (read from `tool_name` in `engine-env.txt`). This enables independent querying: `--source ebpf-dpdk-ovs` vs `--source ebpf-dpdk-testpmd`.

### Epoch-ms timestamps for CDM
Metric samples must use real epoch milliseconds (not 0) to overlap with the CDM period's time window. Timestamps are recorded in `ebpf-dpdk-begin-ms.txt` / `ebpf-dpdk-end-ms.txt` during collection and read during post-processing.

## CDM Integration

### Required schema fields in `cdm.js`
- `metric_desc.names.function` — stores the function name for top-function-pct metrics
- `metric_desc.names.mempool_name` — for DPDK mempool metrics (added previously)

### Metric types emitted
- `top-function-pct` (percentage) — highest CPU% function
- `top1-function-pct` ... `top5-function-pct` (percentage) — ranked by CPU%
- `perf-samples` (count) — total samples collected
- `perf-samples-active` (count) — samples in active traffic window

## Multi-Instance Deployment

Use `"deployment": "opt-in"` with `"opt-tag"` to target specific hosts:
- OVS instance on compute host: `"opt-tag": "has-ovs"`
- testpmd instance on server host: `"opt-tag": "has-testpmd"`

Each endpoint host must declare matching `"tool-opt-in-tags"` in its config.

## Stop Phase Time Budget

The stop script enforces a 120s `MAX_STOP_TIME`:
1. Stop perf (SIGINT + wait up to 30s, force-kill if approaching limit)
2. Run `perf script` (if >15s remaining)
3. Compress with gzip (if >10s remaining)
4. Fallback: compress raw perf.data if perf-script fails

## Testing

- Run post-processor locally: `cd <tool-data-dir> && TOOLBOX_HOME=/opt/crucible/subprojects/core/toolbox python3 /opt/crucible/subprojects/tools/ebpf-dpdk/ebpf-dpdk-post-process`
- Test pmd-discovery: `python3 pmd-discovery.py --target ovs-vswitchd --output json`
- Validate syntax: `python3 -c "import py_compile; py_compile.compile('ebpf-dpdk-post-process', doraise=True)"`
- Run test suite: `TOOLBOX_HOME=/opt/crucible/subprojects/core/toolbox pytest tests/`
- Full integration: `crucible run <run-file.json>` with ebpf-dpdk configured

## Conventions
- Primary branch is `main`
- Standard Bash modelines and 4-space indentation
- Python code follows 4-space indentation with standard modelines
