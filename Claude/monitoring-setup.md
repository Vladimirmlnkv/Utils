# Self-Hosted Monitoring Stack — Setup Reference

## Stack
- **Grafana** — dashboards + log explorer
- **Prometheus** — metrics collection
- **Loki** — log aggregation
- **Promtail** — log shipper (reads PM2/app logs)
- **Node Exporter** — host CPU/RAM/disk
- **postgres_exporter** — Postgres metrics
- **redis_exporter** — Redis metrics

All via Docker Compose at `/opt/monitoring/`.

---

## Directory layout
```
/opt/monitoring/
├── docker-compose.yml
├── prometheus.yml
├── promtail-config.yml
├── .env                          # GRAFANA_PASSWORD=...
├── provisioning/
│   ├── datasources/datasources.yml
│   └── dashboards/dashboards.yml
└── dashboards/
    ├── node-exporter.json        # download from grafana.com id:1860
    ├── redis.json                # grafana.com id:763 or similar
    └── pm2-logs.json             # custom or loki explore
```

---

## `docker-compose.yml`
```yaml
services:
  grafana:
    image: grafana/grafana:latest
    ports:
      - "127.0.0.1:3001:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./provisioning:/etc/grafana/provisioning:ro
      - ./dashboards:/var/lib/grafana/dashboards:ro
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    extra_hosts:
      - "host.docker.internal:host-gateway"   # REQUIRED on Linux
    restart: unless-stopped

  loki:
    image: grafana/loki:latest
    volumes:
      - loki_data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    restart: unless-stopped

  promtail:
    image: grafana/promtail:latest
    volumes:
      - /root/.pm2/logs:/pm2logs:ro            # adjust log path
      - ./promtail-config.yml:/etc/promtail/config.yml:ro
    command: -config.file=/etc/promtail/config.yml
    restart: unless-stopped

  node_exporter:
    image: prom/node-exporter:latest
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped

  postgres_exporter:
    image: prometheuscommunity/postgres-exporter:latest
    network_mode: host                          # needs localhost DB access
    environment:
      - DATA_SOURCE_NAME=postgresql://user:pass@localhost:5432/dbname?sslmode=disable
    restart: unless-stopped

  redis_exporter:
    image: oliver006/redis_exporter:latest
    network_mode: host                          # needs localhost Redis access
    environment:
      - REDIS_ADDR=redis://localhost:6379
    restart: unless-stopped

volumes:
  grafana_data:
  prometheus_data:
  loki_data:
```

---

## `prometheus.yml`
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: node
    static_configs:
      - targets: ['node_exporter:9100']         # container name resolves

  - job_name: postgres
    static_configs:
      - targets: ['host.docker.internal:9187']  # host-network exporter

  - job_name: redis
    static_configs:
      - targets: ['host.docker.internal:9121']  # host-network exporter
```

---

## `promtail-config.yml`
```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: pm2
    static_configs:
      - targets: [localhost]
        labels:
          job: yourapp
          __path__: /pm2logs/yourapp-*.log
```

---

## `provisioning/datasources/datasources.yml`
```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: false
```

## `provisioning/dashboards/dashboards.yml`
```yaml
apiVersion: 1
providers:
  - name: default
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
```

---

## Nginx (subdomain proxy)
```nginx
server {
    server_name monitor.yourdomain.com;
    location / {
        proxy_pass http://localhost:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
    listen 443 ssl;
    # ssl managed by certbot
}
```
```bash
certbot --nginx -d monitor.yourdomain.com
```

---

## CRITICAL: Linux UFW firewall fix

`postgres_exporter` and `redis_exporter` use `network_mode: host` (they need `localhost` DB/Redis access). Prometheus runs in a bridge network and reaches them via `host.docker.internal`. On Linux with UFW, this traffic is **blocked by default**.

Find your monitoring stack's bridge IPs:
```bash
ip addr show | grep '172\.'
# docker0 = default bridge (172.17.0.0/16)
# br-xxxx = compose network (172.18.0.0/16 or similar)
```

Open both for each exporter port:
```bash
ufw allow from 172.17.0.0/16 to any port 9121 proto tcp
ufw allow from 172.17.0.0/16 to any port 9187 proto tcp
ufw allow from 172.18.0.0/16 to any port 9121 proto tcp
ufw allow from 172.18.0.0/16 to any port 9187 proto tcp
```

Also requires `extra_hosts: ["host.docker.internal:host-gateway"]` on the Prometheus service (already in compose above).

Verify Prometheus can reach exporters:
```bash
docker exec <prometheus-container> wget -qO- http://host.docker.internal:9121/metrics | head -3
```

Verify targets are up:
```bash
docker exec <prometheus-container> wget -qO- 'http://localhost:9090/api/v1/targets' \
  | python3 -c "import sys,json; [print(t['labels'].get('job'), t['health']) for t in json.load(sys.stdin)['data']['activeTargets']]"
```

---

## Deploy
```bash
cd /opt/monitoring
echo "GRAFANA_PASSWORD=yourpassword" > .env
docker compose up -d
docker compose ps    # verify all up
```

## Maintenance
```bash
docker compose pull && docker compose up -d   # update images
docker compose logs -f <service>              # debug
docker compose ps                             # status
```
