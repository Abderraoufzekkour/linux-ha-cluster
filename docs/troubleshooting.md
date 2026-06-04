# Troubleshooting Guide
## Linux HA Cluster - Production Operations

---

## 1. Cluster Health Verification

### Check overall cluster status
```bash
sudo pcs status
```

Expected healthy output:
- All nodes listed under Online
- All resources showing Started
- No Failed Resource Actions
- No WARNINGS

### Check Corosync ring status
```bash
sudo corosync-cfgtool -s
```

Expected: all rings showing status ACTIVE. A faulty ring indicates network issues between nodes.

### Check Pacemaker logs
```bash
sudo journalctl -u pacemaker -n 50 --no-pager
```

---

## 2. Resource Failures

### Resource stuck in failed state
```bash
sudo pcs resource cleanup
sudo pcs status
```

This clears failed action history and forces Pacemaker to retry starting the resource.

### Resource not starting on any node
```bash
sudo pcs resource debug-start <resource-name>
```

This attempts to start the resource manually and shows the exact error from the resource agent.

### VirtualIP not reachable from other nodes
This is an OpenStack-specific issue. Verify allowed address pairs on all VM ports:
```bash
openstack port list --network test-network-02
openstack port show <port-id> | grep allowed
```

If the VirtualIP is missing from allowed address pairs, add it:
```bash
openstack port set --allowed-address ip-address=10.10.10.200 <port-id>
```

Run this for all 3 node ports.

### Force resource to specific node
```bash
sudo pcs resource move VirtualIP node01
```

Remove the constraint after migration:
```bash
sudo pcs resource clear VirtualIP
```

---

## 3. DRBD Issues

### Check DRBD replication status
```bash
sudo drbdadm status cluster
```

Healthy output shows:
- role: Primary on active node
- peer role: Secondary on standby
- replication: Established
- peer-disk: UpToDate

### DRBD split-brain recovery
If both nodes show Primary role simultaneously:

On the node to be demoted (sacrifice node):
```bash
sudo drbdadm disconnect cluster
sudo drbdadm secondary cluster
sudo drbdadm connect --discard-my-data cluster
```

On the surviving primary node:
```bash
sudo drbdadm connect cluster
```

### DRBD disk state shows Inconsistent
This occurs after a node crash during replication. Force resync from primary:
```bash
sudo drbdadm primary cluster
sudo drbdadm invalidate-remote cluster
```

### Check DRBD replication traffic
```bash
watch -n1 cat /proc/drbd
```

Shows real-time replication statistics including sync percentage and speed.

---

## 4. Monitoring Issues

### Prometheus not scraping a node
Check if Node Exporter is running on the target node:
```bash
sudo systemctl status node_exporter
curl http://10.10.10.19:9100/metrics | head -5
```

Verify Prometheus targets are up:
```bash
curl http://10.10.10.115:9090/api/v1/targets | python3 -m json.tool | grep health
```

### Alertmanager not receiving alerts
Check Prometheus is configured with Alertmanager:
```bash
curl http://10.10.10.115:9090/api/v1/alertmanagers
```

Check active alerts:
```bash
curl http://10.10.10.115:9093/api/v1/alerts | python3 -m json.tool
```

### Grafana dashboard showing No Data
Verify the Prometheus data source URL is correct:
- Go to Connections > Data Sources > Prometheus
- URL should be http://10.10.10.115:9090
- Click Save and Test

---

## 5. Loki and Log Aggregation Issues

### Promtail not shipping logs
Check Promtail service status:
```bash
sudo systemctl status promtail
sudo journalctl -u promtail -n 20 --no-pager
```

Verify Loki is accepting connections:
```bash
curl http://10.10.10.115:3100/ready
```

Expected response: ready

### Loki failing to start
Check available disk space on node03:
```bash
df -h /tmp
```

Loki stores data in /tmp/loki. If disk is full, clear old chunks:
```bash
sudo rm -rf /tmp/loki/chunks/*
sudo systemctl restart loki
```

---

## 6. Ansible Deployment Issues

### Playbook fails on specific node
Run with increased verbosity to identify the exact task failing:
```bash
ansible-playbook -i ansible/inventory.ini ansible/site.yml -vvv
```

### SSH connectivity issues between control node and cluster nodes
Verify SSH key is present and has correct permissions:
```bash
ls -la ~/.ssh/abderraouf-prod.pem
chmod 600 ~/.ssh/abderraouf-prod.pem
```

Test Ansible connectivity:
```bash
ansible -i ansible/inventory.ini cluster_nodes -m ping
```

### Idempotency check
Run the playbook twice. The second run should show zero changed tasks:
```bash
ansible-playbook -i ansible/inventory.ini ansible/site.yml
```

If tasks still show changed on second run, the role is not idempotent and needs fixing.

---

## 7. Drone CI Pipeline Issues

### Pipeline stuck in pending state
Check if the Drone runner is connected:
```bash
sudo docker ps | grep drone
sudo docker logs drone-runner --tail 20
```

The runner must be running and connected to the Drone server.

### Pipeline failing at Ansible step
Check the pipeline logs in the Drone UI. Common causes:
- SSH key not available inside the pipeline container
- Inventory file pointing to wrong IPs
- Ansible not installed in the pipeline Docker image

---

## 8. Key Commands Reference

| Task | Command |
|---|---|
| Check cluster status | sudo pcs status |
| Check DRBD status | sudo drbdadm status cluster |
| Check all services | sudo systemctl status corosync pacemaker drbd |
| Check Prometheus targets | curl http://10.10.10.115:9090/api/v1/targets |
| Check active alerts | curl http://10.10.10.115:9093/api/v1/alerts |
| Check Loki ready | curl http://10.10.10.115:3100/ready |
| Run Ansible playbook | ansible-playbook -i ansible/inventory.ini ansible/site.yml |
| Check Docker containers | sudo docker ps |
| Restart all cluster services | sudo pcs cluster stop --all && sudo pcs cluster start --all |

---

## Author

Zekkour Abderraouf | Infrastructure & Cloud Engineer
GitHub: Abderraoufzekkour

