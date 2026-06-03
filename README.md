# Linux HA Cluster - Production-Grade Infrastructure

## Overview
A 3-node High Availability cluster built on OpenStack, featuring automatic failover, centralized monitoring, and log aggregation. Built as a portfolio project targeting Linux Sysadmin / Cloud Infrastructure roles.

## Architecture
## Stack
- **HA:** Pacemaker + Corosync + VirtualIP + nginx + HAProxy
- **Automation:** Ansible (roles: common, pacemaker, monitoring, logging)
- **Monitoring:** Prometheus + Grafana + Node Exporter
- **Logging:** Loki + Promtail
- **Infrastructure:** OpenStack (private network, security groups, floating IPs)

## Results
- Failover tested: **1 second RTO** measured by continuous ping
- All 3 nodes monitored in real time via Grafana dashboards
- Centralized log aggregation from all 3 nodes via Loki

## Project Structure## How to Deploy
```bash
git clone https://github.com/Abderraoufzekkour/linux-ha-cluster.git
cd linux-ha-cluster
vim ansible/inventory.ini
ansible-playbook -i ansible/inventory.ini ansible/site.yml
```

## Failover Test Result
Node01 was powered off during a continuous ping to the VirtualIP.
Only 1 packet was lost (icmp_seq=19) before the cluster automatically
moved all resources to node02. Total downtime: 1 second.

## Author
Zekkour Abderraouf - Network, Systems & Telecommunications Engineering
ENSTICP Algiers - 2026
