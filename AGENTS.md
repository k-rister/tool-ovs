# Tool-ovs

## Purpose
Crucible tool for collecting and post-processing Open vSwitch (OVS) and Open Virtual Network (OVN) performance data during benchmark execution. Captures bridge configurations, OpenFlow tables, PMD thread statistics, datapath lookup rates, coverage counters, and memory usage.

## Languages
- Bash: collection and lifecycle scripts (`ovs-collect`, `ovs-start`, `ovs-stop`)
- Python: post-processor (`ovs-post-process.py`)

## Key Files
| File | Purpose |
|------|---------|
| `ovs-collect` | Main collection loop: dumps flows, appctl stats, coverage/memory data at configurable intervals |
| `ovs-start` | Parses `--interval` parameter (default: `3`), launches `ovs-collect` in background |
| `ovs-stop` | Sends SIGTERM to collector, compresses output logs with xz |
| `ovs-post-process.py` | Multiprocess post-processor converting raw OVS data to CDM metrics (`ovs-dpctl`, `ovs-appctl`, `ovs-ofctl`, `ovs-pmd`) |
| `rickshaw.json` | Rickshaw integration: collector scripts, blacklist/whitelist |
| `workshop.json` | Engine image build: compiles OVS 3.5.4 from source |
| `tool-metadata.json` | Machine-readable description and CDM-indexed status (consumed by `crucible tools list`) |
| `multiplex.json` | Parameter validation rules and `defaults` preset for multiplex (mirrors benchmark `multiplex.json`) |

## Configuration
- `--interval <seconds>` — Polling interval in seconds (default: `3`)

## Architecture
- `ovs-start` — Validates `ovs-collect` presence, starts `ovs-collect $interval &`, and stores PID in `ovs-collect-pid.txt`
- `ovs-collect` — Queries `ovs-vsctl` for bridge list, then loops over `ovs-ofctl` (dump-ports, dump-flows) and `ovs-appctl` (dpctl/dump-flows, dpctl/ct-stats-show, dpctl/show, coverage/show, memory/show, upcall/show, dpif-netdev/pmd-perf-show)
- `ovs-stop` — Sends SIGTERM to `ovs-collect`, verifies termination, and compresses raw data files (`ofctl*.txt`, `appctl*.txt`, `pmd-stats-clear.stdouterr.txt`) with xz
- `ovs-post-process.py` — Spawns parallel worker processes for each data stream:
  - `conntrack_post_process` -> `ovs-dpctl:ct-stats-show`
  - `dpctl_memory_show_process` -> `ovs-appctl:mem-show`
  - `ofctl_port_counters` -> `ovs-ofctl:packets-sec`, `ovs-ofctl:Gbps`, `ovs-ofctl:errors-sec`
  - `appctl_dpif_netdev_pmd_perf_show` -> `ovs-pmd:datapath-hits-sec`, `ovs-pmd:kpps`, `ovs-pmd:pmd-busy`, `ovs-pmd:pmd-idle`
  - `dpctl_datapath_stats` -> `ovs-dpctl:flows-count`, `ovs-dpctl:lookups-sec`, `ovs-dpctl:masks-sec`, `ovs-dpctl:cache-sec`
  - `upcall_stats` -> `ovs-appctl:upcall-flow`, `ovs-appctl:upcall-flow-avg`, `ovs-appctl:upcall-flow-max`, `ovs-appctl:upcall-flow-limit`, `ovs-appctl:upcall-flow-dump-duration-ms`
  - `dpctl_dump_flows` -> `ovs-dpctl:ufid-new-flows-sec`, `ovs-dpctl:ufid-expired-flows-sec`, `ovs-dpctl:ufid-packets-sec`, `ovs-dpctl:ufid-Gbps`

## Testing
- Run post-processor locally: `cd <tool-data-dir> && TOOLBOX_HOME=/opt/crucible/subprojects/core/toolbox python3 /opt/crucible/subprojects/tools/ovs/ovs-post-process.py`
- Validate syntax: `python3 -c "import py_compile; py_compile.compile('ovs-post-process.py', doraise=True)"`
- Full integration: `crucible run <run-file.json>` with ovs tool configured on OVS-enabled endpoint

## Conventions
- Primary branch is `master`
- Runs as a profiler tool on master/worker/profiler/compute roles, blocked on client/server
- Standard Bash modelines and 4-space indentation
- Python code follows 4-space indentation with standard modelines
