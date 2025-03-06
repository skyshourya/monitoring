# monitoring
Nginx Monitoring with Prometheus, Grafana, and Alertmanager

# Nginx Monitoring with Prometheus, Grafana, and Alertmanager

## Overview
This setup provides real-time monitoring for an Nginx server using 'Nginx_exporter', Prometheus, and Grafana. It also includes Alertmanager for sending notifications to Slack when Nginx or the EC2 instance goes down.

## Components Used
- **Prometheus**: Collects and stores metrics.
- **Nginx Exporter**: Exposes Nginx metrics to Prometheus.
- **Grafana**: Visualizes metrics with dashboards.
- **Alertmanager**: Handles alerts and sends notifications to Slack.

## Prerequisites
- An Ubuntu-based EC2 instance.
- Docker and Docker Compose installed.
- Nginx installed and running.

## Setup Instructions

### 1. Install Dependencies
```bash
sudo apt update
sudo apt install -y docker.io docker-compose
```

### 2. Clone the Repository
```bash
git clone https://github.com/your-repo/nginx-monitoring.git
cd nginx-monitoring
```

### 3. Configure Nginx for Monitoring
Modify the `nginx.conf` file to enable stub_status:
```nginx
server {
    listen 80;
	server_name 13.201.81.35; # EC2 instance IP

	location /metrics {
            proxy_pass http://13.201.57.151:9090/metrics;  # Corrected service URL
            allow 13.201.57.151;
            allow 13.201.81.35;  # Allow access from your IP or specific range
            deny all;  # Deny all other access
        } 
        
        location /nginx_status {
            stub_status;
            allow 13.201.57.151;  # Allow Prometheus server to access the status page
            allow 13.201.81.35;
	    allow 127.0.0.1;
	    deny all;
	   
        }

}
```
Restart Nginx:
```bash
sudo systemctl restart nginx
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
        channel: '#sunny'
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

**Maintainer:** Shourya

For any issues, please open a GitHub issue.


