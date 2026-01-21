# 🔥 Ember

**A quiet system for revealing hidden heat and load.**

Ember is a standalone Linux host observability stack designed to diagnose fan noise, thermal spikes, and system bottlenecks using host and container metrics. Built for Mini PCs and homelab environments where understanding thermal behavior and resource utilization is critical.

![Ember Logo](assets/ember.png)

## Features

- **Thermal Monitoring**: Track CPU temperatures, thermal zones, and fan speeds via hwmon
- **CPU Analysis**: Per-core utilization, iowait detection, load averages
- **Memory & Swap**: Real-time memory breakdown and swap usage tracking
- **Disk I/O**: Throughput, IOPS, and filesystem usage monitoring
- **Network**: Bandwidth utilization, errors, and packet drops
- **Container Insights**: Top containers by CPU/memory, restart detection
- **Process Visibility**: Per-process resource consumption (optional)

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Linux Host                               │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │ node_exporter│  │   cAdvisor   │  │   process-exporter     │ │
│  │   :9100      │  │    :8080     │  │        :9256           │ │
│  └──────┬───────┘  └──────┬───────┘  └───────────┬────────────┘ │
│         │                 │                      │              │
│         └────────────┬────┴──────────────────────┘              │
│                      ▼                                          │
│              ┌───────────────┐                                  │
│              │  Prometheus   │──── 15 day retention             │
│              │    :9090      │                                  │
│              └───────┬───────┘                                  │
│                      │                                          │
│                      ▼                                          │
│              ┌───────────────┐                                  │
│              │    Grafana    │──── Auto-provisioned dashboards  │
│              │    :3000      │                                  │
│              └───────────────┘                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Prerequisites

- **Docker** >= 20.10
- **Docker Compose** >= 2.0 (or docker-compose v1.29+)
- **Linux host** with access to `/proc`, `/sys`, and `/dev`
- **lm-sensors** (recommended for temperature metrics)

### Verify Docker Installation

```bash
docker --version
docker compose version
```

## Quick Start

### 1. Clone or Create the Project

```bash
cd /path/to/ember
```

### 2. Configure Environment

Copy the example environment file and optionally generate a new secure password:

```bash
# Use the provided .env (has a pre-generated secure password)
# OR generate your own:
cp .env.example .env
echo "GF_SECURITY_ADMIN_PASSWORD=$(openssl rand -base64 24)" > .env
```

### 3. Start the Stack

```bash
docker compose up -d
```

### 4. Verify All Services Are Running

```bash
docker compose ps
```

Expected output shows all 5 services as "healthy" or "running":
- `ember-prometheus`
- `ember-grafana`
- `ember-node-exporter`
- `ember-cadvisor`
- `ember-process-exporter`

## Accessing the Interfaces

| Service    | URL                        | Default Credentials |
|------------|----------------------------|---------------------|
| Grafana    | http://localhost:3000      | admin / (see .env)  |
| Prometheus | http://localhost:9090      | -                   |

> **Note**: All services bind to `127.0.0.1` (localhost only) by default for security.

## Verification Steps

### Check Prometheus Targets

1. Open http://localhost:9090/targets
2. Verify all targets show **UP** status:
   - `node-exporter` (1/1 up)
   - `cadvisor` (1/1 up)
   - `process-exporter` (1/1 up)
   - `prometheus` (1/1 up)

### Check Grafana Dashboards

1. Open http://localhost:3000
2. Login with `admin` and the password from your `.env` file
3. Navigate to **Dashboards** in the left sidebar
4. Verify two dashboards are present:
   - **Host Health + Thermals**
   - **Containers Overview**

### Verify Metrics Collection

In Prometheus (http://localhost:9090/graph), try these queries:

```promql
# CPU temperature (requires lm-sensors)
node_hwmon_temp_celsius

# CPU usage
rate(node_cpu_seconds_total{mode="user"}[1m])

# Container memory
container_memory_usage_bytes{name!=""}

# Process CPU
namedprocess_namegroup_cpu_seconds_total
```

## Enabling Temperature Metrics

Temperature metrics require **lm-sensors** to be installed and configured on the host.

### Install lm-sensors

**Debian/Ubuntu:**
```bash
sudo apt update
sudo apt install lm-sensors
```

**Fedora/RHEL:**
```bash
sudo dnf install lm_sensors
```

**Arch Linux:**
```bash
sudo pacman -S lm_sensors
```

### Detect Sensors

Run the sensor detection wizard:

```bash
sudo sensors-detect
```

- Answer **YES** to probe for various sensor chips
- Answer **YES** to add modules to `/etc/modules` when prompted
- Reboot or load modules manually:

```bash
sudo systemctl restart systemd-modules-load.service
```

### Verify Sensors

```bash
sensors
```

Expected output shows temperature readings:

```
coretemp-isa-0000
Adapter: ISA adapter
Package id 0:  +45.0°C  (high = +100.0°C, crit = +100.0°C)
Core 0:        +43.0°C  (high = +100.0°C, crit = +100.0°C)
Core 1:        +44.0°C  (high = +100.0°C, crit = +100.0°C)
```

### Check hwmon Files

Verify sensor data is exposed in sysfs:

```bash
ls /sys/class/hwmon/
cat /sys/class/hwmon/hwmon*/temp*_input
```

### Restart node_exporter

After configuring lm-sensors, restart the stack to pick up new sensors:

```bash
docker compose restart node-exporter
```

## If Temperature Metrics Are Missing

1. **Check if hwmon is exposed:**
   ```bash
   ls -la /sys/class/hwmon/
   ```

2. **Verify node_exporter can read hwmon:**
   ```bash
   curl -s http://localhost:9100/metrics | grep hwmon
   ```

3. **Check for thermal_zone metrics (alternative):**
   ```bash
   curl -s http://localhost:9100/metrics | grep thermal_zone
   ```

4. **Ensure kernel modules are loaded:**
   ```bash
   lsmod | grep -E 'coretemp|k10temp|nct|it87'
   ```

5. **Common issues:**
   - Some Mini PCs don't expose fan RPM via hwmon
   - Virtual machines typically don't have hwmon sensors
   - BIOS/UEFI settings may disable sensor reporting

## Commands Reference

### Start Stack
```bash
docker compose up -d
```

### Stop Stack
```bash
docker compose down
```

### View Logs
```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f prometheus
docker compose logs -f grafana
docker compose logs -f node-exporter
```

### Restart Services
```bash
docker compose restart
```

### Update Images
```bash
docker compose pull
docker compose up -d
```

### Check Resource Usage
```bash
docker stats
```

### Remove Everything (including data)
```bash
docker compose down -v
```

## Data Persistence

Data is stored in Docker named volumes:

| Volume            | Purpose                    |
|-------------------|----------------------------|
| `prometheus_data` | Prometheus TSDB (15 days)  |
| `grafana_data`    | Grafana config & state     |

To backup volumes:

```bash
docker run --rm -v ember_prometheus_data:/data -v $(pwd):/backup alpine tar czf /backup/prometheus-backup.tar.gz /data
docker run --rm -v ember_grafana_data:/data -v $(pwd):/backup alpine tar czf /backup/grafana-backup.tar.gz /data
```

## Security Notes

### Localhost Binding

All services bind to `127.0.0.1` by default, making them accessible only from the local machine. This is intentional for security.

**To expose externally** (not recommended without additional security):

Edit `docker-compose.yml` and change port bindings:
```yaml
ports:
  - "0.0.0.0:3000:3000"  # Exposes to all interfaces
```

### Secrets Management

- **Never commit `.env` to version control** (it's in `.gitignore`)
- The `.env.example` file shows the required format without real secrets
- Generate strong passwords: `openssl rand -base64 24`

### Container Privileges

- `node-exporter`: Runs with host PID namespace for accurate process metrics
- `cadvisor`: Runs privileged to access container metrics
- `process-exporter`: Runs privileged to read `/proc`

These are required for accurate metrics collection.

## Troubleshooting

### host.docker.internal Not Resolving

On Linux, `host.docker.internal` requires explicit configuration. The `docker-compose.yml` includes:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

If you still have issues:

1. Verify Docker version >= 20.10
2. Check the host gateway IP:
   ```bash
   docker run --rm alpine ip route | grep default
   ```
3. Manually specify the host IP in `prometheus.yml` if needed

### Prometheus Can't Scrape Targets

1. Check target status: http://localhost:9090/targets
2. Verify containers are running: `docker compose ps`
3. Check container logs: `docker compose logs <service>`
4. Test connectivity from Prometheus container:
   ```bash
   docker compose exec prometheus wget -qO- http://node-exporter:9100/metrics | head
   ```

### Grafana Dashboard Shows "No Data"

1. Verify Prometheus datasource: **Grafana → Connections → Data sources → Prometheus**
2. Check if Prometheus has data: http://localhost:9090/graph
3. Ensure time range is appropriate (default: last 1 hour)
4. Check for metric name changes between versions

### cAdvisor Memory Issues

cAdvisor can be memory-intensive. To limit:

```yaml
cadvisor:
  deploy:
    resources:
      limits:
        memory: 512M
```

### High Disk Usage

Prometheus stores 15 days of data. To reduce:

1. Edit `docker-compose.yml`:
   ```yaml
   command:
     - '--storage.tsdb.retention.time=7d'
   ```
2. Restart Prometheus:
   ```bash
   docker compose restart prometheus
   ```

### process-exporter High Cardinality

If you have many unique processes, edit `process-exporter/process-exporter.yml` to group more aggressively or exclude noisy processes.

## Customization

### Changing Scrape Intervals

Edit `prometheus/prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'node-exporter'
    scrape_interval: 10s  # Change from 5s
```

### Adding Custom Dashboards

1. Create JSON dashboard file in `grafana/dashboards/`
2. Dashboards are auto-loaded within 30 seconds
3. Or restart Grafana: `docker compose restart grafana`

### Modifying Retention

Edit retention in `docker-compose.yml`:

```yaml
prometheus:
  command:
    - '--storage.tsdb.retention.time=30d'  # 30 days instead of 15
```

## Stack Versions

| Component        | Version   |
|------------------|-----------|
| Prometheus       | v2.51.0   |
| Grafana          | 10.4.1    |
| node_exporter    | v1.7.0    |
| cAdvisor         | v0.47.2   |
| process-exporter | 0.8.2     |

## License

This project is provided as-is for personal and educational use.

---

**Ember** — *Revealing the hidden heat in your system* 🔥
