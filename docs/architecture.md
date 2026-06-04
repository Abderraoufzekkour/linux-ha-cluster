# Architecture Documentation
## Linux HA Cluster - Production-Grade Infrastructure Platform

## 1. Executive Summary

This document describes the architecture of a 3-node High Availability infrastructure platform deployed on OpenStack. The platform is designed to meet the following production requirements:

- RTO (Recovery Time Objective): Less than 30 seconds (achieved: 1 second)
- RPO (Recovery Point Objective): Zero data loss
- Availability target: 99.9% uptime
- Automation: 100% infrastructure-as-code via Ansible
- Observability: Full metrics, logs, and alerting coverage

## 2. Node Roles

| Node | Private IP | Public IP | Primary Role |
|---|---|---|---|
| node01 | 10.10.10.19 | 197.140.32.170 | HA quorum voter + DRBD primary |
| node02 | 10.10.10.173 | 197.140.32.59 | HA standby + DRBD secondary |
| node03 | 10.10.10.115 | 197.140.32.230 | HA active node + Control plane |

## 3. High Availability Layer

Corosync sends heartbeat messages every 1 second between all nodes. When a node fails to respond for 3 consecutive heartbeats, Corosync declares it dead and notifies Pacemaker.

Pacemaker manages three colocated resources that always move together:
- VirtualIP 10.10.10.200 (IPaddr2 resource agent)
- nginx web server (systemd resource agent)
- haproxy load balancer (systemd resource agent)

Startup ordering: VirtualIP starts first, then nginx, then haproxy.

Measured RTO: 1 second verified by continuous ping test.

## 4. Data Replication Layer

DRBD Protocol C synchronous replication between node01 and node02. A write is only acknowledged after it has been committed to disk on both nodes simultaneously. This guarantees zero RPO.

Verified by writing a timestamped file on node01, promoting node02 to primary, and reading the identical file from node02 without writing anything there manually.

## 5. Observability Stack

Prometheus scrapes metrics from all 3 nodes every 15 seconds via Node Exporter on port 9100. Grafana provides real-time dashboards with per-node drill-down.

Four Alertmanager alert rules:
- NodeDown: fires after 30 seconds of node unavailability (severity: critical)
- HighCPUUsage: fires when CPU exceeds 85% for 2 minutes (severity: warning)
- HighDiskUsage: fires when disk exceeds 85% for 5 minutes (severity: warning)
- HighMemoryUsage: fires when memory exceeds 85% for 2 minutes (severity: warning)

Promtail ships logs from all 3 nodes to Loki on node03. All logs are queryable from Grafana using LogQL without SSH access to individual nodes.

## 6. Automation Layer

All infrastructure is provisioned via Ansible roles: common, pacemaker, monitoring, logging, alertmanager, drbd, gitea.

Gitea provides self-hosted Git on node03 port 3001. Drone CI server and runner run as Docker containers on node03 port 3002. Every push to the main branch triggers an automated pipeline build implementing GitOps principles.

## 7. Critical Network Issue and Resolution

OpenStack Neutron port security drops packets from IP addresses not assigned to a VM port. The VirtualIP 10.10.10.200 is managed by Pacemaker and not known to OpenStack, so traffic was silently dropped with no error from Pacemaker.

Resolution: allowed address pairs configured on all 3 VM ports via OpenStack Neutron API:
openstack port set --allowed-address ip-address=10.10.10.200 <port-id>

## 8. Disaster Recovery

- Single node failure: handled automatically by Pacemaker. RTO 1 second. RPO 0.
- Dual node failure: cluster enters safe mode. Manual intervention required.
- Complete cluster failure: full rebuild from Ansible playbooks in 15 minutes.

## 9. Author

Zekkour Abderraouf | Infrastructure & Cloud Engineer
GitHub: Abderraoufzekkour
Email: a_zekkour@inptic.edu.dz
