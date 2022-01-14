---
title: 安裝Prometheus
tags: tutorials
disqus: hackmd
---

# 下載 https://prometheus.io/download/#prometheus
```
wget https://
tar xvfz filename
```
# 新增user
```
groupadd prometheus
useradd -g prometheus -M -s /bin/false prometheus

sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus

sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool

sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
```


# 設定systemd
```
sudo vi /etc/systemd/system/prometheus.service

[Unit]
Description=Prometheus Time Series Collection and Processing Server
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries \
    --web.external-url=http://172.16.67.32:9090 \
    --web.enable-admin-api \
    --web.enable-lifecycle  #熱重啟curl -X POST localhost:9090/-/reload
[Install]
WantedBy=multi-user.target
```



# vim prometheus.yml檔案
```
#my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
#scrape_timeout is set to the global default (10s).

#Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - 127.0.0.1:9093

#Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  - "/etc/prometheus/alert_rules/*.yml"
   #- "first_rules.yml"
   #- "second_rules.yml"

 #A scrape configuration containing exactly one endpoint to scrape:
 #Here it's Prometheus itself.
scrape_configs:
  #The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    #metrics_path defaults to '/metrics'
    #scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'telegraf'
    static_configs:
    - targets: ['172.16.67.32:9273']

  - job_name: 'windows'
    scrape_interval: 15s
    scrape_timeout: 12s
    metrics_path: /metrics
    file_sd_configs:
    - files: ['/etc/prometheus/windows/*.yml']

systemctl daemon-reload
systemctl enable prometheus.service
systemctl start prometheus.service
```

# 安裝Alertmanager

```
wget https://
tar xvfz filename
```
# vim alertmanager.yml 檔案
```
global:
  resolve_timeout: 15s
route:
  group_by: ['instance', 'alertname']
  group_wait: 5s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'sam-test'
receivers:
- name: 'sam-test'
  webhook_configs:
  - url: 'http://localhost:2000/sam-test'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```
# 啟動systemd
```
[Unit]
Description=Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
WorkingDirectory=/etc/alertmanager/
ExecStart=/etc/alertmanager/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml


[Install]
WantedBy=multi-user.target
```
# 安裝Prometheus-msteams
下載binary檔案與default-message-card.tmpl -
https://github.com/prometheus-msteams/prometheus-msteams#getting-started-quickstart

# 啟動systemd
```
[Unit]
Description=msteams
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/etc/prometheus/prometheus-msteams/prometheus-msteams-linux-amd64 -config-file /etc/prometheus/prometheus-msteams/config.yml \
        -template-file /etc/prometheus/prometheus-msteams/default-message-card.tmpl

[Install]
WantedBy=multi-user.target
```
