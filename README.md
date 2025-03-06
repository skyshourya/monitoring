### Nginx Monitoring with Prometheus, Grafana, and Alertmanager

## Overview
This setup provides real-time monitoring for an Nginx server using `nginx_exporter`, Prometheus, and Grafana. It also includes Alertmanager for sending notifications to Slack when Nginx or the EC2 instance goes down.

## Components Used
- **Prometheus**: Collects and stores metrics.
- **Nginx Exporter**: Exposes Nginx metrics to Prometheus.
- **Grafana**: Visualizes metrics with dashboards.
- **Alertmanager**: Handles alerts and sends notifications to Slack.

## Prerequisites
- An Ubuntu-based EC2 instance.
- Docker and Docker Compose installed.
- **Nginx installed and running.***

## Setup Instructions

### 1. Install Dependencies
```bash
sudo apt update
sudo apt install -y docker.io docker-compose
```

### 2. Docker Compose Configuration on the host where nginx is running

Create a `docker-compose.yml` file with the following content:
```bash
version: '3.8'
services:
  nginx-exporter:
    image: nginx/nginx-prometheus-exporter:latest
    container_name: nginx-exporter
    command:
      - "--nginx.scrape-uri=http://<HOST_IP>/nginx_status"
    ports:
      - "9113:9113"
    restart: always
```

### 3. Configure Nginx for Monitoring
Modify the `nginx.conf` file to enable stub_status:
```nginx
# inside the http {} 
server {
    listen 80;
	server_name xx.xx.xx.xx; # EC2 instance IP

	location /metrics {
            proxy_pass http://13.201.57.151:9090/metrics;  # Corrected service URL
            allow xx.xx.xx.xx;
            allow xx.xx.xx.xx;  # Allow access from your IP or specific range
            deny all;  # Deny all other access
        } 
        
        location /nginx_status {
            stub_status;
            allow xx.xx.xx.xx;  # Allow Prometheus server to access the status page
            allow xx.xx.xx.xx;   # IP's which are allowed to access the status page 
	    allow <local_host_IP>;
	    deny all;
	   
        }

}
```
Restart Nginx:
```bash
sudo systemctl restart nginx
```
### 4. Docker Compose Configuration on the host where Prometheus, Grafana, and Alertmanager will be running.
Create a `docker-compose.yml` file with the following content:
```bash
version: '3.8'

services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alert.rules.yml:/etc/prometheus/alert.rules.yml
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
    ports:
      - "9090:9090"
    depends_on:
      - alertmanager

  alertmanager:
    image: prom/alertmanager
    container_name: alertmanager
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    command:
      - "--config.file=/etc/alertmanager/alertmanager.yml"
    ports:
      - "9093:9093"

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
    grafana-data:

```

### 4. Start the Monitoring Stack
```bash
docker-compose up -d
```

### 5. Verify Prometheus Targets
Open Prometheus in a browser:
```
http://<EC2-IP>:9090
```
Navigate to `Status -> Targets` and check if `nginx_exporter` is up.

### 6. Access Grafana
```
http://<EC2-IP>:3000
```
Login using:
- **Username:** admin
- **Password:** admin

Add Prometheus as a data source and import an Nginx monitoring dashboard.

### 7. Alerting Setup
Alerts are defined in `alert.rules.yml`:
```yaml
  - alert: NginxDown
    expr: nginx_up == 0
    for: 20s
    labels:
      severity: critical
    annotations:
      summary: "Nginx is down on {{ $labels.instance }}"
      description: "Nginx process on {{ $labels.instance }} is unreachable for over 20 seconds."
```
Slack notifications are configured in `alertmanager.yml`:
```yaml
receivers:
  - name: 'slack-notifications'
    slack_configs:
      - send_resolved: true
        channel: '#NAME'
        api_url: 'https://hooks.slack.com/services/YOUR_SLACK_WEBHOOK_URL'
```

### 8. Test Alerts
Stop the Nginx container and check if Slack receives an alert:
```bash
docker stop nginx
```

## Conclusion
This setup ensures that your Nginx server is continuously monitored, and any downtime triggers an alert on Slack. You can extend this setup to monitor additional services as needed.

---

**Maintainer:** Shourya Yadav


