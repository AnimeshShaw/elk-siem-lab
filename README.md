# ELK SIEM Lab — Windows Docker Setup Guide

## Prerequisites

1. **Docker Desktop** (Windows) with WSL2 backend enabled

   - Download: https://www.docker.com/products/docker-desktop/
   - In Settings → General → enable "Use WSL 2 based engine"
2. **WSL2 memory fix** (required for Elasticsearch)
   Open PowerShell as Administrator and run:

   ```powershell
   wsl -d docker-desktop sysctl -w vm.max_map_count=262144
   ```

   To make it permanent, create/edit `%USERPROFILE%\.wslconfig`:

   ```ini
   [wsl2]
   kernelCommandLine = sysctl.vm.max_map_count=262144
   ```
3. **Docker resources** — In Docker Desktop → Settings → Resources:

   - Memory: minimum **4 GB** (6 GB recommended)
   - CPUs: 2+

---

## Lab Directory Structure

```
elk-siem-lab/
├── docker-compose.yml
├── logstash/
│   ├── pipeline/
│   │   └── siem-lab.conf        ← Logstash pipeline (parse + enrich)
│   └── config/
│       └── logstash.yml
├── filebeat/
│   └── filebeat.yml
└── sample-logs/
    └── system.log               ← Pre-loaded sample logs with security events
```

---

## Start the Lab

```powershell
# From the elk-siem-lab directory:
docker compose up -d

# Watch startup logs (wait ~60-90 seconds for full init):
docker compose logs -f
```

### Verify all containers are up:

```powershell
docker compose ps
```

Expected output:

```
NAME              STATUS
elasticsearch     Up (healthy)
logstash          Up
kibana            Up
filebeat          Up
```

---

## Access the Stack

| Service       | URL                   | Notes          |
| ------------- | --------------------- | -------------- |
| Kibana (UI)   | http://localhost:5601 | Main dashboard |
| Elasticsearch | http://localhost:9200 | REST API       |
| Logstash API  | http://localhost:9600 | Pipeline stats |

---

## Lab Exercises

### Exercise 1 — Verify log ingestion

Check that Filebeat shipped the sample logs:

```powershell
curl http://localhost:9200/siem-logs-*/_count
```

You should see a `count > 0`.

### Exercise 2 — Explore in Kibana

1. Open http://localhost:5601
2. Go to **Menu → Discover**
3. Create a data view: index pattern = `siem-logs-*`, time field = `@timestamp`
4. Explore the parsed fields: `syslog_host`, `severity`, tags like `security_alert`

### Exercise 3 — Send a live log via TCP (simulate a syslog source)

```powershell
# In PowerShell — send a test syslog line to Logstash TCP input
$client = New-Object System.Net.Sockets.TcpClient("localhost", 5000)
$stream = $client.GetStream()
$msg = "Apr 17 09:00:00 testhost sshd[9999]: Failed password for root from 1.2.3.4 port 22 ssh2`n"
$bytes = [System.Text.Encoding]::UTF8.GetBytes($msg)
$stream.Write($bytes, 0, $bytes.Length)
$client.Close()
```

Then search for it in Kibana Discover.

### Exercise 4 — Build a security dashboard in Kibana

1. Go to **Dashboards → Create dashboard**
2. Add a **Lens** visualization:
   - Bar chart: count of events by `syslog_host`
   - Pie chart: `tags` breakdown (spot `security_alert` events)
3. Save the dashboard as "SIEM Overview"

## Useful Commands

```powershell
# Stop the lab (preserves Elasticsearch data)
docker compose stop

# Full teardown including data
docker compose down -v

# View Logstash pipeline logs
docker logs logstash

# Re-index after editing sample-logs
docker compose restart filebeat

# Check index mappings
curl http://localhost:9200/siem-logs-*/_mapping
```

---

## Troubleshooting

| Issue                                     | Fix                                                   |
| ----------------------------------------- | ----------------------------------------------------- |
| Elasticsearch exits with code 78          | Run the `vm.max_map_count` WSL2 fix above           |
| Kibana shows "Kibana server is not ready" | Wait 90s; Elasticsearch may still be starting         |
| No logs in Kibana                         | Run `docker logs filebeat` to check shipping errors |
| Port already in use                       | Check `netstat -ano` and stop conflicting services  |

---

## Architecture Reference

```
sample-logs/system.log
        │
        ▼
   [Filebeat]          ← agent: tails files, ships to Logstash
        │ (port 5044)
        ▼
   [Logstash]          ← parse (Grok), normalize, enrich, tag alerts
        │
        ▼
[Elasticsearch]        ← stores + indexes logs (index: siem-logs-YYYY.MM.DD)
        │
        ▼
   [Kibana]            ← search, visualize, dashboard, alerting
```

**Core SIEM concepts demonstrated:**

- **Collection**: Filebeat tails log files and ships them
- **Normalization**: Grok patterns parse unstructured text into structured fields
- **Correlation**: Logstash filter tags events matching attack patterns (brute-force, SQLi, malware)
- **Visualization**: Kibana dashboards surface the attack timeline
