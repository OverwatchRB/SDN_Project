# SDN Project — Hardware Switch Placement in Multicast Trees

**Authors:** Vuchi Gnanasagar (CS22B092), Rohan Bagati (CS22B082), Ashwin Kilingar (ME22B111)

**Source Code:** https://github.com/OverwatchRB/SDN_Project

---

## Overview

This project investigates how hardware switch placement in an overlay multicast tree affects
delivery latency and fairness (Delivery Window Size). We run three experiments using
Mininet-based emulation.

- **Experiment 1:** Compare host-proxy vs BMv2-switch replication, and evaluate a
  hold-and-release mechanism to reduce DWS.
- **Experiment 2:** Evaluate three hardware switch placement strategies (Root, Subtree Cover,
  Uniform) under a limited hardware budget, using fast AF_PACKET proxies vs slow tc-netem proxies.
- **Experiment 3:** P4/BMv2 overlay tree with a One Big Switch (OBS) fabric.

---

## Experiment 1

```
cd Experiment1
```

### Baseline host proxy (no helpers)
```bash
sudo python3 baseline_host.py --d 4 --cycles 1000 --gap 1 --no-cli
```

### Baseline BMv2 switch (no helpers)
```bash
sudo python3 baseline_switch.py --d 4 --cycles 1000 --gap 1 \
    --multicast-json multicast_int.json --no-cli
```

### Hold-and-release host proxy
```bash
# Busy-wait mode
sudo python3 holdrelease_host.py --d 4 --target-latency 5.0 \
    --mode busywait --cycles 1000 --gap 1 --no-cli

# Sleep mode
sudo python3 holdrelease_host.py --d 4 --target-latency 5.0 \
    --mode sleep --cycles 1000 --gap 1 --no-cli
```

### Hold-and-release BMv2 switch
```bash
# Busy-wait mode
sudo python3 holdrelease_switch.py --d 4 --target-latency 5.0 \
    --mode busywait --cycles 1000 --gap 1 \
    --multicast-json multicast_int.json --no-cli

# Sleep mode
sudo python3 holdrelease_switch.py --d 4 --target-latency 5.0 \
    --mode sleep --cycles 1000 --gap 1 \
    --multicast-json multicast_int.json --no-cli
```

### Plotting
Run all d values (2 to 10) first, then:
```bash
python3 plot_host_comparison.py --base . --out host_comparison.png
python3 plot_switch_comparison.py --base . --out switch_comparison.png
```

---

## Experiment 2

```
cd Experiment2
```

### Run a single configuration
```bash
sudo python3 hybrid_topo_host_model.py \
    --h 3 --d 4 --k 5 \
    --placement root \
    --cycles 1000 --gap 2.0 \
    --no-cli
```

### Key arguments
| Argument | Description |
|----------|-------------|
| `--h` | Tree height (used: 3) |
| `--d` | Fanout per node (used: 3 or 4) |
| `--k` | Number of hardware switch nodes (0 to 13 for d=3, 0 to 21 for d=4) |
| `--placement` | `root`, `subtree_cover`, or `uniform` |
| `--hedging-factor` | Hedging parameter H (0 = off, 1 = on) |
| `--cycles` | Number of packets (used: 1000) |
| `--gap` | Inter-packet gap in ms (used: 2.0ms = 500pps) |
| `--spin-count` | Spin loop iterations for slow proxy (used: 40000) |
| `--delay` | tc netem delay in ms for slow proxy (used: 1.0) |
| `--variance` | tc netem variance in ms for slow proxy (used: 0.1) |

### With hedging
```bash
sudo python3 hybrid_topo_host_model_only.py \
    --h 3 --d 4 --k 5 \
    --placement root \
    --hedging-factor 1 \
    --cycles 1000 --gap 2.0 \
    --no-cli
```

### Plotting
```bash
# Mean DWS and OML
python3 plot_mean.py --base . --out mean_h3d4.png --h-filter 3 --d-filter 4 --H-filter 0
python3 plot_mean.py --base . --out mean_h3d3.png --h-filter 3 --d-filter 3 --H-filter 0

# p90 DWS and OML
python3 plot_p90.py --base . --out p90_h3d4.png --h-filter 3 --d-filter 4 --H-filter 0
python3 plot_p90.py --base . --out p90_h3d3.png --h-filter 3 --d-filter 3 --H-filter 0

# With hedging (H=1)
python3 plot_mean.py --base . --out mean_h3d4_H1.png --h-filter 3 --d-filter 4 --H-filter 1
```

---

## Experiment 3

```
cd Experiment3
```

### Run a single configuration
```bash
sudo python3 hybrid_topo_final.py \
    --h 3 --d 4 --k 5 \
    --placement root \
    --cycles 100 --gap 50.0 \
    --multicast-json multicast_int.json \
    --obs-json ip_fwd.json \
    --no-cli
```

### Key arguments
| Argument | Description |
|----------|-------------|
| `--h` | Tree height |
| `--d` | Fanout per node |
| `--k` | Number of BMv2 hardware switch nodes |
| `--placement` | `root`, `subtree_cover`, or `uniform` |
| `--cycles` | Number of packets (use low values, e.g. 100, due to OBS bottleneck) |
| `--gap` | Inter-packet gap in ms (use >= 50ms to avoid packet drops) |
| `--multicast-json` | Path to compiled P4 JSON for multicast switches |
| `--obs-json` | Path to compiled P4 JSON for OBS fabric switch |

> **Note:** The OBS switch becomes a bottleneck at high packet rates. Keep `--gap >= 50`
> and `--cycles <= 100` to avoid drops. This is a known limitation discussed in the report.

### Plotting
```bash
python3 plot_mean_final.py --base . --out exp3_h3d4.png --h-filter 3 --d-filter 4
python3 plot_mean_final.py --base . --out exp3_h3d3.png --h-filter 3 --d-filter 3
```

---

## Metrics

- **DWS (Delivery Window Size):** `max(recv_ts) - min(recv_ts)` across all receivers
  for the same message. Measures simultaneity of delivery.
- **OML (Overall Multicast Latency):** `max(e2e_latency)` across all receivers for the
  same message. Measures worst-case end-to-end delay.

Both are computed offline per `msg_id` across all receiver CSV logs.
