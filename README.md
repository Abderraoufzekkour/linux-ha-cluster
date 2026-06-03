# Linux HA Cluster - Production-Grade Infrastructure Platform

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Ansible](https://img.shields.io/badge/Ansible-2.10-red.svg)
![Pacemaker](https://img.shields.io/badge/Pacemaker-2.1-green.svg)
![Prometheus](https://img.shields.io/badge/Prometheus-2.49-orange.svg)
![Grafana](https://img.shields.io/badge/Grafana-13.0-yellow.svg)

## Overview

This project implements a **production-grade 3-node High Availability cluster** deployed on OpenStack. The infrastructure provides automatic failover, zero-manual-intervention recovery, real-time monitoring, and centralized log aggregation.

The entire stack is provisioned and configured using **Ansible**, following Infrastructure-as-Code best practices. Every component is version-controlled, documented, and reproducible from a single command.

## Architecture
node01 (10.10.10.19)   - Quorum voter
node02 (10.10.10.173)  - Standby node  
node03 (10.10.10.115)  - Active node + Prometheus + Grafana + Loki
VirtualIP (10.10.10.200) - Floating between nodes
Network: test-network-02 (10.10.10.0/24) on OpenStack Neutron

## Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| High Availability | Pacemaker + Corosync | Cluster management and heartbeating |
| Virtual IP | IPaddr2 resource agent | Floating IP between nodes |
| Web Server | nginx | HTTP service managed by Pacemaker |
| Load Balancer | HAProxy | Traffic distribution |
| Automation | Ansible | Full infrastructure provisioning |
| Monitoring | Prometheus + Node Exporter | Metrics collection from all nodes |
| Dashboards | Grafana | Real-time visualization |
| Log Aggregation | Loki + Promtail | Centralized logging from all nodes |
| Infrastructure | OpenStack | Private cloud environment |

## Failover Results

| Metric | Value |
|---|---|
| RTO (Recovery Time Objective) | **1 second** |
| RPO (Recovery Point Objective) | 0 (no data loss) |
| Failover method | Automatic (no human intervention) |
| Detection time | ~3 seconds (Corosync heartbeat) |
| Resource migration | VirtualIP + nginx + HAProxy move together |

## Screenshots

### Cluster Failover - node01 Offline
![Failover](01-cluster-failover-node01-offline.png)

### Ping Test - 1 Second RTO
![Ping](02-failover-ping-1-second-rto.png)

### Cluster Recovery - All Nodes Online
![Recovery](03-cluster-recovery-all-nodes-online.png)

### Grafana Monitoring - node01
![Grafana node01](04-grafana-monitoring-node01.png)

### Grafana Monitoring - node02
![Grafana node02](05-grafana-monitoring-node02.png)

### Loki Centralized Logging
![Loki](06-loki-centralized-logging.png)

## Project Structure
ansible/
  inventory.ini        - Node definitions and SSH config
  site.yml             - Main playbook entry point
  roles/
    common/            - Base packages, hostname, hosts file
    pacemaker/         - Corosync + Pacemaker + HA resources
    monitoring/        - Prometheus + Grafana + Node Exporter
    logging/           - Loki + Promtail log aggregation
screenshots/           - Project evidence
README.md

## How to Deploy

### Prerequisites
- 3 Ubuntu 22.04 VMs on the same network
- Ansible installed on the control node
- SSH key access to all nodes

### Deploy

```bash
git clone https://github.com/Abderraoufzekkour/linux-ha-cluster.git
cd linux-ha-cluster
vim ansible/inventory.ini
ansible-playbook -i ansible/inventory.ini ansible/site.yml
```

## Challenges and Solutions

| Challenge | Solution |
|---|---|
| OpenStack blocking VirtualIP traffic | Added allowed address pairs on all 3 ports via Neutron API |
| Wi-Fi blocking SSH from local PC | Used management VM as jump host inside ICOSNET |
| Loki failing to start | Created missing compactor directory and updated config |
| Ubuntu cloud image no password | Used cloud-init user data script to inject credentials |

## Author

**Zekkour Abderraouf** | Infrastructure & Cloud Engineer

- GitHub: [Abderraoufzekkour](https://github.com/Abderraoufzekkour)
