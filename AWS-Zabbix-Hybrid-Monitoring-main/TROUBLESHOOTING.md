# 🔧 Troubleshooting Guide

Common issues encountered during deployment and how to fix them.

---

## 1. ZBX Indicator Stays Red / Host Unreachable

**Symptom:** The host appears in Zabbix but the ZBX icon is red or grey.

**Checklist:**
- [ ] Is the agent service actually running?
  ```bash
  # Linux
  sudo systemctl status zabbix-agent

  # Windows (PowerShell)
  Get-Service "Zabbix Agent"
  ```
- [ ] Does the agent config point to the correct **private IP** of the Zabbix server?
  ```bash
  grep "^Server=" /etc/zabbix/zabbix_agentd.conf
  # Expected: Server=10.0.1.X  (not a public IP)
  ```
- [ ] Does the AWS Security Group for the agent instance allow **inbound TCP 10050** from the subnet `10.0.1.0/24`?
- [ ] Does the AWS Security Group for the Zabbix Server allow **outbound TCP 10050** (or is it open by default)?
- [ ] Test connectivity manually from the Zabbix Server:
  ```bash
  nc -zv <AGENT_PRIVATE_IP> 10050
  ```

---

## 2. Zabbix Web UI Not Loading (port 80)

**Symptom:** Browser times out or returns "connection refused" on `http://<PUBLIC_IP>`.

**Checklist:**
- [ ] Are all 3 Docker containers running?
  ```bash
  docker ps
  # Should show: zabbix-server, zabbix-web-nginx-mysql, mysql-server
  ```
- [ ] Did you wait at least 60 seconds after `docker compose up -d`? MySQL takes time to initialize.
- [ ] Check container logs for errors:
  ```bash
  docker compose logs zabbix-web-nginx-mysql
  docker compose logs mysql-server
  ```
- [ ] Does the Security Group for the Zabbix Server allow **inbound TCP 80** from `0.0.0.0/0`?

---

## 3. Docker Compose MySQL Connection Error

**Symptom:** `zabbix-server` container keeps restarting; logs show "Can't connect to MySQL server".

**Fix:** MySQL wasn't ready when Zabbix tried to connect. Restart the stack:
```bash
docker compose down
docker compose up -d
```
If the problem persists, check that the passwords in `docker-compose.yml` (or `.env`) are **identical** across all three services.

---

## 4. `docker compose` Command Not Found

**Symptom:** Running `docker compose` returns "command not found".

**Fix:** On Ubuntu 24.04, the v2 plugin is a separate package:
```bash
sudo apt install docker-compose-v2
# Use: docker compose (with a space), NOT docker-compose (with a hyphen)
```

---

## 5. Windows Agent Not Connecting

**Symptom:** Windows host is unreachable from Zabbix.

**Checklist:**
- [ ] Is Windows Firewall allowing port 10050?
  ```powershell
  Get-NetFirewallRule -DisplayName "Zabbix Agent"
  # If missing, run:
  New-NetFirewallRule -DisplayName "Zabbix Agent" -Direction Inbound -LocalPort 10050 -Protocol TCP -Action Allow
  ```
- [ ] Is the Zabbix Agent service running?
  ```powershell
  Get-Service "Zabbix Agent"
  Start-Service "Zabbix Agent"
  ```
- [ ] Check the agent log for errors:
  ```
  C:\Program Files\Zabbix Agent\zabbix_agentd.log
  ```
- [ ] Verify the server IP was set correctly during installation. To change it, edit:
  ```
  C:\Program Files\Zabbix Agent\zabbix_agentd.conf
  ```
  Then restart the service.

---

## 6. Hostname Mismatch Error

**Symptom:** Agent connects but Zabbix shows "host not found" or metrics don't appear.

**Fix:** The `Hostname=` value in `zabbix_agentd.conf` must **exactly match** the host name entered in the Zabbix Web UI (case-sensitive).

```bash
# Linux: check current hostname in config
grep "^Hostname=" /etc/zabbix/zabbix_agentd.conf
```

---

## 7. AWS Learner Lab Session Expired

**Symptom:** EC2 instances stopped; public IPs have changed after restarting the lab.

**Note:** AWS Learner Lab assigns **new public IPs** each session. Private IPs stay the same.
- Update your browser bookmark with the new public IP of the Zabbix Server.
- Agent configs use **private IPs** — these don't change, so no reconfiguration needed.

---

## Useful Commands Reference

```bash
# Check all Zabbix containers
docker ps -a

# Restart the full stack
docker compose down && docker compose up -d

# Live logs from all containers
docker compose logs -f

# Test agent port from server
nc -zv <AGENT_PRIVATE_IP> 10050

# Check agent service status (Linux)
sudo systemctl status zabbix-agent

# Restart agent (Linux)
sudo systemctl restart zabbix-agent
```
