# NATWORK Deception & Adversarial Experiment Dataset

## Overview

This dataset contains measurements from two experiment series on a Kubernetes-based Open5GS 5G core testbed, instrumented with honeytoken-based deception assets in the UDR network function:

- **Baseline experiments** evaluate the performance impact of deployed deception assets during normal 5G operation. A fixed D-ITG traffic profile is executed with and without honey assets, across honeyfile densities and traffic volumes, to measure resource and network overhead.
- **Campaign experiments** evaluate the deception assets under a post-compromise scenario. A fixed Apache Caldera adversary campaign is executed against the target UDR to determine whether deployed honey assets are discovered and interacted with, and whether this is observable to the monitoring system (Tetragon/eBPF).

The same adversary campaign, target UDR, deception deployment procedure, monitoring configuration, and alert rules are used across all five evaluated honeyfile densities (0%, 25%, 50%, 75%, 90%), with **one campaign run per density** (no replication). Honeyfile generation uses a fixed random seed for reproducibility.

All files are partitioned using Hive-style `key=value` directories and are provided as Apache Parquet, directly queryable with DuckDB, pandas/pyarrow, or Spark without a separate schema definition.

```sql
-- Example: query every file at once via glob + hive partitioning
SELECT * FROM read_parquet('campaign_tetragon/**/*.parquet', hive_partitioning=true) LIMIT 5;
```

## File-by-file description

### Baseline

| File                                                         | Contents                                                                                                                                                                                           | Partition keys                        |
|--------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------|
| `baseline_prometheus.parquet`                                  | 18 Prometheus metrics (CPU, memory, restart count, target-up status per NF; VM memory; UPF RX/TX throughput), long format with a `metric_name` column                                                | `density`, `volume`, `run`                  |
| `baseline_ditg_flow_summary.parquet`                           | Per-flow `ITGDec` summary statistics: delay (min/max/avg/stddev), jitter, bitrate, packet rate, packets dropped (count + %), loss-burst size. One row per D-ITG flow per direction (uplink/downlink) | `density`, `workload`, `run`, `direction`     |
| `baseline_ditg_console_events.parquet`                         | D-ITG sender/receiver console lifecycle events (port listen/finish, flow start/finish, uplink device binding). Event-level, not performance data                                                   | `density`, `volume`, `run`, `direction`, `role` |
| `baseline_downlink/uplink_profile.parquet`                     | D-ITG down-/uplink flow configuration (destination IP/port, protocol, packet rate, packet size, duration) as sent to `ITGSend`, one row per configured flow                                          | `density`, `volume`, `run`                  |
| `baseline_healthy_ues.parquet`, `baseline_unhealthy_ues.parquet` | Per-run UE health status                                                                                                                                                                           | `density`, `volume`, `run`                  |
| `baseline_ue-map.parquet`                                      | UE-to-identifier mapping for the run                                                                                                                                                               | `density`, `volume`, `run`                  |

### Campaign

| File                                  | Contents                                                                                                                                                                                                                                                                                            | Partition keys                                     |
|---------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| `campaign_tetragon/`                    | Tetragon/eBPF kernel telemetry from the UDR pod: process executions (`EXEC`), file reads (`READ`), file writes (`WRITE`). Columns: `ts`, `event_type`, `pid`, `ppid` (EXEC only), `binary`, `target` (command args for EXEC, file path for READ/WRITE)                                                                | `density`, `run` (physical Hive-partitioned directory) |
| `campaign_caldera_events.parquet`       | Adversary emulation log from Apache Caldera: one row per executed attack step, with command, plaintext command, delegated/collected/finished timestamps, PID, agent metadata (paw, username, host, privilege), ability metadata, operation metadata, and MITRE ATT&CK tactic/technique/technique ID | `density`, `run`                                       |
| `campaign_caldera_tetragon_map.parquet` | Pre-built correlation between each Caldera attack step and the Tetragon event(s) it produced: caldera start/end time, tactic, ability name, plaintext command, earliest matching Tetragon timestamp, and the full matched Tetragon log line                                                         | `density`, `run`                                       |
| `campaign_discovery_listing.parquet`    | Parsed directory-listing output captured during the adversary's discovery phase (`caldera-discovery.log`): one row per `(directory, entry)` pair observed by the attacker                                                                                                                               | `density`, `run`                                       |

## How the files relate

- `campaign_caldera_events.parquet` provides the full per-step Caldera record (timestamps, PID, agent and operation metadata, MITRE ATT&CK tactic/technique/technique ID) for every executed attack step, verified to be correctly scoped per density.
- `campaign_tetragon` is the defender's-side ground truth of what actually executed or was accessed on the UDR pod.
- `campaign_caldera_tetragon_map.parquet` is the authoritative correlation between Caldera attack steps and the Tetragon events they produced**.** It was built directly by the experiment author using plaintext command matching within a time window, and should be treated as the primary reference for "which attacker action triggered which observed system event." 
- `campaign_discovery_listing.parquet` shows what was visible to the attacker during reconnaissance, independent of both of the above

## Known limitations

- **Single run per density.** Each of the five honeyfile density configurations was evaluated with exactly one campaign run and (for baseline) one run per density/volume combination. There is no within-configuration replication, so observed differences across densities should not be interpreted as statistically validated effects without corroborating evidence.
- **No per-flow time-series performance data in baseline, only per-flow aggregates.**`baseline_ditg_flow_summary.parquet` contains `ITGDec`-computed summary statistics per flow (mean/min/max delay, jitter, bitrate, loss), not the underlying time-series matrices. Raw `.dat` time-series files exist in the original experiment tree but are not included in this submission.
- **D-ITG data does not exist for campaign runs.** Campaign experiments measure detection, not network performance, so no traffic-profile or quality-of-service data exists for the `campaign` scenario by design.
- **The honeyfile density=0 configuration is not deception-free.** It retains two fixed behavioral executable traps (deployed identically at every density); only the honeyfile count scales with the density parameter. Fixed honey environment variables and honey binaries/processes are proof-of-concept assets present at every density level.

## Reproducibility

Testbed deployment (k3s, Open5GS, UERANSIM), deception-asset deployment tooling, Caldera adversary profiles, Tetragon tracing policies, and experiment protocol scripts are available at: <https://github.com/thphan21/Deceptive-Assets-for-Orchestrated-5G-Networks>