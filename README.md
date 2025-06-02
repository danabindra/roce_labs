# Lossless RDMA Fabric Configuration – RoCEv2

## Overview
This document outlines the configuration steps and tuning parameters used to achieve a lossless RDMA fabric for training workloads using RoCEv2. The environment includes VXLAN VRFs, ECN, PFC, and benchmarking via `perftest`.

## Architecture Summary
- Fabric Type: Layer 2/3 Ethernet with VXLAN
- Transport: RDMA over Converged Ethernet (RoCEv2)
- Topology: Spine-Leaf with 100G/200G interconnects
- Workload: Distributed GPU training (PyTorch/Horovod)
- NICs: Mellanox ConnectX-6/7
- Switch OS: SONiC / Cumulus Linux
- Kubernetes: GPU nodes orchestrated by Cluster API and Talos Linux
- Telemetry: Prometheus + custom `rocm-smi` exporter + `ethtool` counters

## Key Tuning 

### 1. Priority Flow Control (PFC) – Per switch + NIC

```bash
# Enable PFC on switches (SONiC/Cumulus)
dcbtool sc eth0 pfc e:1
```

```bash
# NIC tuning (Mellanox)
mlnx_qos -i eth0 --pfc 1 --pfc-prio 3
ethtool -A eth0 rx on tx on autoneg on
```

- Priority 3 is used for RDMA traffic (CoS mapping)
- Enable PFC watchdog to avoid deadlock

### 2. Explicit Congestion Notification (ECN)

```bash
# Enable ECN on switch queues
echo 1 > /sys/module/mlx5_core/parameters/ecn_enable
tc qdisc add dev eth0 root fq_codel ecn
```

- Enable ECN marking on switch queues with tail drop thresholds
- Ensure GPU pods have DSCP or CoS tags for ECN mapping

### 3. VXLAN with RDMA Support

```bash
# Kernel parameter
echo 1 > /sys/module/mlx5_core/parameters/enable_vxlan_rdma
```

- Configure switch overlay routes with vxlan interfaces
- Ensure VLAN → VXLAN mappings preserve DSCP/CoS bits
