# Security Monitoring Setup Guide

## Overview

This guide provides step-by-step instructions for setting up comprehensive security monitoring using the latest versions of Prometheus (v2.47.0), Grafana (v10.2.0), and the Elastic Stack (v8.11.0).

## Prerequisites

- Docker Engine v24.0+ and Docker Compose v2.20+
- At least 8GB RAM and 50GB storage
- Linux host with kernel 4.0+ for optimal performance
- Network access to monitored systems

## Quick Start

### 1. Environment Setup

```bash
# Clone the repository
git clone https://github.com/securedbyfajobi/security-monitoring.git
cd security-monitoring

# Create environment file
cp .env.example .env

# Generate secure passwords
export ELASTIC_PASSWORD=$(openssl rand -base64 32)
export KIBANA_PASSWORD=$(openssl rand -base64 32)
export GRAFANA_PASSWORD=$(openssl rand -base64 32)
export ENCRYPTION_KEY=$(openssl rand -base64 32)
export GRAFANA_SECRET_KEY=$(openssl rand -base64 32)

# Update .env file
cat > .env << EOF
ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
KIBANA_PASSWORD=${KIBANA_PASSWORD}
GRAFANA_PASSWORD=${GRAFANA_PASSWORD}
ENCRYPTION_KEY=${ENCRYPTION_KEY}
GRAFANA_SECRET_KEY=${GRAFANA_SECRET_KEY}
EOF
```

### 2. Certificate Generation

```bash
# Create certificates directory
mkdir -p certs/{ca,elasticsearch,kibana,grafana}

# Generate CA certificate
openssl genrsa -out certs/ca/ca.key 4096
openssl req -new -x509 -days 3650 -key certs/ca/ca.key -out certs/ca/ca.crt \
  -subj "/CN=Security-Monitoring-CA"

# Generate Elasticsearch certificates
openssl genrsa -out certs/elasticsearch/elasticsearch.key 4096
openssl req -new -key certs/elasticsearch/elasticsearch.key \
  -out certs/elasticsearch/elasticsearch.csr \
  -subj "/CN=elasticsearch"
openssl x509 -req -in certs/elasticsearch/elasticsearch.csr \
  -CA certs/ca/ca.crt -CAkey certs/ca/ca.key -CAcreateserial \
  -out certs/elasticsearch/elasticsearch.crt -days 365

# Generate Kibana certificates
openssl genrsa -out certs/kibana/kibana.key 4096
openssl req -new -key certs/kibana/kibana.key \
  -out certs/kibana/kibana.csr \
  -subj "/CN=kibana"
openssl x509 -req -in certs/kibana/kibana.csr \
  -CA certs/ca/ca.crt -CAkey certs/ca/ca.key -CAcreateserial \
  -out certs/kibana/kibana.crt -days 365

# Generate Grafana certificates
openssl genrsa -out certs/grafana/grafana.key 4096
openssl req -new -key certs/grafana/grafana.key \
  -out certs/grafana/grafana.csr \
  -subj "/CN=grafana"
openssl x509 -req -in certs/grafana/grafana.csr \
  -CA certs/ca/ca.crt -CAkey certs/ca/ca.key -CAcreateserial \
  -out certs/grafana/grafana.crt -days 365

# Set proper permissions
chmod 600 certs/**/*.key
chmod 644 certs/**/*.crt
```

### 3. Deploy the Stack

```bash
# Start the monitoring stack
docker-compose up -d

# Wait for services to be ready
echo "Waiting for Elasticsearch to be ready..."
until curl -s --cacert certs/ca/ca.crt \
  -u "elastic:${ELASTIC_PASSWORD}" \
  https://localhost:9200/_cluster/health; do
  sleep 10
done

echo "Monitoring stack is ready!"
```

### 4. Configure Kibana Users

```bash
# Set Kibana system password
docker exec elasticsearch /usr/share/elasticsearch/bin/elasticsearch-reset-password \
  -u kibana_system -p ${KIBANA_PASSWORD} --batch

# Create additional users for monitoring
docker exec elasticsearch /usr/share/elasticsearch/bin/elasticsearch-users useradd \
  security_analyst -p $(openssl rand -base64 12) -r superuser

docker exec elasticsearch /usr/share/elasticsearch/bin/elasticsearch-users useradd \
  readonly_user -p $(openssl rand -base64 12) -r viewer
```

## Configuration

### Prometheus Configuration

Create `prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "rules/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'elasticsearch'
    static_configs:
      - targets: ['elasticsearch-exporter:9114']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'security-metrics'
    static_configs:
      - targets: ['security-exporter:8080']
    scrape_interval: 10s
    metrics_path: /metrics

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

### Filebeat Configuration

Create `filebeat/filebeat.yml`:

```yaml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/auth.log
    - /var/log/secure
    - /var/log/syslog
  fields:
    logtype: system
  fields_under_root: true

- type: log
  enabled: true
  paths:
    - /var/log/nginx/access.log
    - /var/log/apache2/access.log
  fields:
    logtype: web
  fields_under_root: true

- type: docker
  enabled: true
  containers.ids:
    - "*"
  fields:
    logtype: container
  fields_under_root: true

processors:
- add_host_metadata:
    when.not.contains.tags: forwarded

- add_docker_metadata: ~

- add_kubernetes_metadata:
    host: ${NODE_NAME}
    matchers:
    - logs_path:
        logs_path: "/var/log/containers/"

output.elasticsearch:
  hosts: ["https://elasticsearch:9200"]
  username: "elastic"
  password: "${ELASTICSEARCH_PASSWORD}"
  ssl:
    certificate_authorities: ["/usr/share/filebeat/certs/ca/ca.crt"]

setup.kibana:
  host: "https://kibana:5601"
  username: "elastic"
  password: "${ELASTICSEARCH_PASSWORD}"
  ssl:
    certificate_authorities: ["/usr/share/filebeat/certs/ca/ca.crt"]

setup.ilm.enabled: true
setup.ilm.rollover_alias: "filebeat"
setup.ilm.pattern: "{now/d}-000001"
setup.ilm.policy: |
  {
    "policy": {
      "phases": {
        "hot": {
          "actions": {
            "rollover": {
              "max_size": "50GB",
              "max_age": "1d"
            }
          }
        },
        "delete": {
          "min_age": "30d",
          "actions": {
            "delete": {}
          }
        }
      }
    }
  }
```

### Metricbeat Configuration

Create `metricbeat/metricbeat.yml`:

```yaml
metricbeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false

metricbeat.modules:
- module: system
  metricsets:
    - cpu
    - load
    - memory
    - network
    - process
    - process_summary
    - socket_summary
  enabled: true
  period: 10s
  processes: ['.*']

- module: docker
  metricsets:
    - container
    - cpu
    - diskio
    - info
    - memory
    - network
  hosts: ["unix:///var/run/docker.sock"]
  period: 10s
  enabled: true

- module: kubernetes
  metricsets:
    - node
    - system
    - pod
    - container
    - volume
  period: 10s
  enabled: true
  hosts: ["kube-state-metrics:8080"]

- module: elasticsearch
  metricsets:
    - node
    - node_stats
    - cluster_stats
  period: 10s
  hosts: ["https://elasticsearch:9200"]
  username: "elastic"
  password: "${ELASTICSEARCH_PASSWORD}"
  ssl:
    certificate_authorities: ["/usr/share/metricbeat/certs/ca/ca.crt"]

output.elasticsearch:
  hosts: ["https://elasticsearch:9200"]
  username: "elastic"
  password: "${ELASTICSEARCH_PASSWORD}"
  ssl:
    certificate_authorities: ["/usr/share/metricbeat/certs/ca/ca.crt"]

setup.kibana:
  host: "https://kibana:5601"
  username: "elastic"
  password: "${ELASTICSEARCH_PASSWORD}"
  ssl:
    certificate_authorities: ["/usr/share/metricbeat/certs/ca/ca.crt"]
```

## Security Configuration

### 1. Network Security

```bash
# Configure firewall rules
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 443/tcp     # HTTPS
sudo ufw allow 9090/tcp    # Prometheus (internal only)
sudo ufw allow 3000/tcp    # Grafana (HTTPS)
sudo ufw allow 5601/tcp    # Kibana (HTTPS)
sudo ufw enable

# Configure Docker network isolation
docker network create --driver bridge \
  --subnet=172.20.0.0/16 \
  --opt com.docker.network.bridge.name=security-monitoring \
  security-monitoring
```

### 2. Authentication and Authorization

```bash
# Create Grafana service account
docker exec grafana grafana-cli admin create-service-account \
  --name security-automation

# Configure RBAC for Elasticsearch
cat > users_roles.yml << EOF
superuser:
  - security_analyst

viewer:
  - readonly_user
  - kibana_user

monitoring:
  - beats_system
  - logstash_system
EOF

docker cp users_roles.yml elasticsearch:/usr/share/elasticsearch/config/
```

### 3. Data Encryption

All communications are encrypted using TLS 1.3 with:
- 4096-bit RSA keys
- SHA-256 signatures
- Perfect Forward Secrecy
- Certificate rotation every 365 days

## Dashboard Import

### Grafana Dashboards

```bash
# Import security monitoring dashboards
curl -X POST \
  -H "Content-Type: application/json" \
  -u "admin:${GRAFANA_PASSWORD}" \
  -d @grafana-dashboards/security-overview.json \
  https://localhost:3000/api/dashboards/db

curl -X POST \
  -H "Content-Type: application/json" \
  -u "admin:${GRAFANA_PASSWORD}" \
  -d @grafana-dashboards/threat-detection.json \
  https://localhost:3000/api/dashboards/db
```

### Kibana Dashboards

```bash
# Import Kibana saved objects
curl -X POST \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -u "elastic:${ELASTIC_PASSWORD}" \
  -d @kibana/saved_objects.ndjson \
  https://localhost:5601/api/saved_objects/_import
```

## Alerting Configuration

### Prometheus Alertmanager

Create `alertmanager/alertmanager.yml`:

```yaml
global:
  smtp_smarthost: 'localhost:587'
  smtp_from: 'security-alerts@company.com'

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'

  routes:
  - match:
      severity: critical
    receiver: 'security-team'
    group_wait: 5s
    repeat_interval: 30m

  - match:
      category: compliance
    receiver: 'compliance-team'

receivers:
- name: 'web.hook'
  webhook_configs:
  - url: 'http://localhost:5001/'

- name: 'security-team'
  email_configs:
  - to: 'security-team@company.com'
    subject: 'CRITICAL: Security Alert'
    body: |
      Alert: {{ .GroupLabels.alertname }}
      Summary: {{ .CommonAnnotations.summary }}
      Description: {{ .CommonAnnotations.description }}

  slack_configs:
  - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
    channel: '#security-alerts'
    title: 'Security Alert'
    text: '{{ .CommonAnnotations.summary }}'

- name: 'compliance-team'
  email_configs:
  - to: 'compliance@company.com'
    subject: 'Compliance Alert'
```

## Performance Tuning

### Elasticsearch Optimization

```bash
# Configure Elasticsearch for security monitoring
cat > elasticsearch/elasticsearch.yml << EOF
cluster.name: security-monitoring
node.name: security-node-1

# Memory settings
bootstrap.memory_lock: true

# Security settings
xpack.security.enabled: true
xpack.security.http.ssl.enabled: true
xpack.security.transport.ssl.enabled: true

# Performance settings
indices.memory.index_buffer_size: 30%
thread_pool.write.queue_size: 1000
thread_pool.search.queue_size: 1000

# Index lifecycle management
action.auto_create_index: .security*,.monitoring*,.watches*,.triggered_watches*,.watcher-history*,.ml*
EOF
```

### Resource Allocation

```bash
# Set JVM heap size for Elasticsearch
echo "ES_JAVA_OPTS=-Xms4g -Xmx4g" >> /etc/environment

# Configure system limits
cat >> /etc/security/limits.conf << EOF
elasticsearch soft memlock unlimited
elasticsearch hard memlock unlimited
elasticsearch soft nofile 65536
elasticsearch hard nofile 65536
EOF
```

## Maintenance

### Daily Tasks

```bash
#!/bin/bash
# daily-maintenance.sh

# Rotate logs
docker exec elasticsearch curator --config /opt/curator/config.yml \
  /opt/curator/actions.yml

# Update threat intelligence feeds
python3 threat-intelligence/update-feeds.py

# Generate compliance reports
python3 compliance/generate-daily-report.py

# Health checks
bash scripts/health-check.sh
```

### Weekly Tasks

```bash
#!/bin/bash
# weekly-maintenance.sh

# Certificate expiry check
openssl x509 -in certs/elasticsearch/elasticsearch.crt -noout -dates

# Index optimization
curl -X POST "https://localhost:9200/_optimize?max_num_segments=1" \
  -u "elastic:${ELASTIC_PASSWORD}"

# Backup configurations
tar -czf backups/config-$(date +%Y%m%d).tar.gz \
  prometheus/ grafana/ elk-configs/

# Security scan
bash scripts/security-scan.sh
```

## Troubleshooting

### Common Issues

1. **Elasticsearch startup failure**
   ```bash
   # Check disk space
   df -h

   # Check memory limits
   cat /proc/meminfo | grep MemAvailable

   # Verify certificates
   openssl verify -CAfile certs/ca/ca.crt certs/elasticsearch/elasticsearch.crt
   ```

2. **High memory usage**
   ```bash
   # Monitor memory usage
   docker stats

   # Adjust JVM heap
   echo "ES_JAVA_OPTS=-Xms2g -Xmx2g" > .env
   ```

3. **Network connectivity issues**
   ```bash
   # Check network configuration
   docker network ls
   docker network inspect security-monitoring

   # Test connectivity
   curl -k https://localhost:9200/_cluster/health
   ```

## Security Best Practices

1. **Regular Updates**
   - Monitor for security patches
   - Update Docker images monthly
   - Test updates in staging first

2. **Access Control**
   - Use strong passwords (32+ characters)
   - Enable MFA where possible
   - Regular access reviews

3. **Monitoring**
   - Monitor the monitoring system
   - Set up meta-alerts for system health
   - Regular backup validation

## Support

For issues and questions:
- Documentation: `/docs/`
- GitHub Issues: [Repository Issues](../../issues)
- Security Contact: security@company.com

---

**Last Updated**: December 2023
**Version**: 2.0
**Author**: Security Engineering Team