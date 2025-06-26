# Prometheus 스택 구축

## 🎯 개요

온프렘 쿠버네티스 환경에서 Prometheus, Grafana, AlertManager를 활용한 종합 모니터링 시스템을 구축하는 실전 가이드입니다.

## 🏗️ 모니터링 아키텍처

```
┌─────────────────────────────────────────────────────┐
│                   Grafana                           │
│            (Visualization & Dashboards)            │
└─────────────────┬───────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────┐
│                Prometheus                           │
│             (Metrics Storage & Query)              │
└─────────────────┬───────────────────────────────────┘
                  │
    ┌─────────────┼─────────────┐
    │             │             │
┌───▼───┐    ┌───▼────┐    ┌───▼────┐
│Node   │    │kube-   │    │Custom  │
│Exporter│   │state-  │    │App     │
│       │    │metrics │    │Metrics │
└───────┘    └────────┘    └────────┘
```

## 📦 Helm을 이용한 설치

### Helm 설치

```bash
# Helm 설치
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 버전 확인
helm version

# Prometheus 커뮤니티 차트 저장소 추가
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### kube-prometheus-stack 설치

```yaml
# prometheus-values.yaml
# Prometheus, Grafana, AlertManager 통합 설치

prometheus:
  prometheusSpec:
    # 온프렘 환경에서의 데이터 보존 기간
    retention: 30d
    retentionSize: 50GB
    
    # 스토리지 설정
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: "local-storage"  # 실제 SC명으로 변경
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    
    # 리소스 설정
    resources:
      requests:
        memory: 2Gi
        cpu: 500m
      limits:
        memory: 4Gi
        cpu: 1000m
    
    # 외부 라벨 추가
    externalLabels:
      cluster: "onprem-k8s"
      datacenter: "dc1"
    
    # 스크래핑 간격
    scrapeInterval: 30s
    evaluationInterval: 30s
    
    # 추가 스크래핑 설정
    additionalScrapeConfigs:
      - job_name: 'kubernetes-nodes-custom'
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
          - source_labels: [__address__]
            regex: '(.*):10250'
            target_label: __address__
            replacement: '${1}:9100'

grafana:
  # 관리자 비밀번호 설정
  adminPassword: "admin123!@#"  # 실제 환경에서는 Secret 사용
  
  # 영구 스토리지 설정
  persistence:
    enabled: true
    storageClassName: "local-storage"
    size: 10Gi
  
  # 서비스 타입 설정 (온프렘)
  service:
    type: NodePort
    nodePort: 30300
  
  # Grafana 설정
  grafana.ini:
    server:
      root_url: "http://grafana.company.local"  # 실제 도메인으로 변경
    security:
      allow_embedding: true
    users:
      allow_sign_up: false
    auth:
      disable_login_form: false
  
  # 대시보드 자동 import
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
      - name: 'default'
        orgId: 1
        folder: ''
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/default
  
  # 기본 대시보드
  dashboards:
    default:
      kubernetes-cluster:
        gnetId: 7249
        revision: 1
        datasource: Prometheus
      node-exporter:
        gnetId: 1860
        revision: 27
        datasource: Prometheus
      kubernetes-pods:
        gnetId: 6417
        revision: 1
        datasource: Prometheus

alertmanager:
  alertmanagerSpec:
    # 스토리지 설정
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: "local-storage"
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
    
    # 외부 접근 설정
    externalUrl: "http://alertmanager.company.local"
    
    # 리소스 제한
    resources:
      requests:
        memory: 256Mi
        cpu: 100m
      limits:
        memory: 512Mi
        cpu: 200m

# Node Exporter 설정
nodeExporter:
  enabled: true
  
# kube-state-metrics 설정  
kubeStateMetrics:
  enabled: true

# CoreDNS 모니터링 활성화
coreDns:
  enabled: true

# etcd 모니터링 활성화 (kubeadm 환경)
kubeEtcd:
  enabled: true
  endpoints:
    - 192.168.1.10  # 마스터 노드 IP
    - 192.168.1.11
    - 192.168.1.12
  service:
    enabled: true
    port: 2379
    targetPort: 2379
```

### 설치 실행

```bash
# 모니터링 네임스페이스 생성
kubectl create namespace monitoring

# Prometheus 스택 설치
helm install prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-values.yaml

# 설치 상태 확인
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

## 🔧 추가 Exporter 설치

### Node Exporter (베어메탈 서버용)

```bash
# 베어메탈 서버에 Node Exporter 설치
# 각 물리 서버에서 실행

# 사용자 생성
useradd --no-create-home --shell /bin/false node_exporter

# 바이너리 다운로드
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar xvfz node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
chown node_exporter:node_exporter /usr/local/bin/node_exporter

# 서비스 파일 생성
cat <<EOF | sudo tee /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
  --collector.systemd \
  --collector.processes \
  --collector.interrupts

[Install]
WantedBy=multi-user.target
EOF

# 서비스 시작
systemctl daemon-reload
systemctl enable node_exporter
systemctl start node_exporter

# 상태 확인
systemctl status node_exporter
curl http://localhost:9100/metrics
```

### 하드웨어 모니터링 (IPMI)

```yaml
# ipmi-exporter.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ipmi-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: ipmi-exporter
  template:
    metadata:
      labels:
        app: ipmi-exporter
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: ipmi-exporter
        image: prometheuscommunity/ipmi-exporter:v1.6.1
        ports:
        - containerPort: 9290
          hostPort: 9290
        args:
        - "--config.file=/etc/ipmi_exporter/config.yaml"
        securityContext:
          privileged: true
        volumeMounts:
        - name: dev
          mountPath: /dev
        - name: config
          mountPath: /etc/ipmi_exporter
      volumes:
      - name: dev
        hostPath:
          path: /dev
      - name: config
        configMap:
          name: ipmi-exporter-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: ipmi-exporter-config
  namespace: monitoring
data:
  config.yaml: |
    modules:
      default:
        collectors:
        - ipmi
        - dcmi
        - bmc
        exclude_sensor_ids:
        - 2
```

## 📊 커스텀 메트릭 설정

### 애플리케이션 메트릭

```yaml
# custom-app-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: custom-app-monitor
  namespace: monitoring
  labels:
    app: custom-app
    release: prometheus-stack
spec:
  selector:
    matchLabels:
      app: custom-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
    scheme: http
---
apiVersion: v1
kind: Service
metadata:
  name: custom-app-metrics
  namespace: default
  labels:
    app: custom-app
spec:
  selector:
    app: custom-app
  ports:
  - name: metrics
    port: 8080
    targetPort: 8080
```

### JMX 메트릭 (Java 애플리케이션)

```yaml
# jmx-exporter.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jmx-exporter-config
  namespace: monitoring
data:
  config.yaml: |
    rules:
    - pattern: ".*"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jmx-exporter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jmx-exporter
  template:
    metadata:
      labels:
        app: jmx-exporter
    spec:
      containers:
      - name: jmx-exporter
        image: sscaling/jmx-prometheus-exporter:0.17.2
        ports:
        - containerPort: 5556
        env:
        - name: CONFIG_YML
          value: "/etc/jmx-exporter/config.yaml"
        - name: JVM_OPTS
          value: "-Xmx512m"
        volumeMounts:
        - name: config
          mountPath: /etc/jmx-exporter
      volumes:
      - name: config
        configMap:
          name: jmx-exporter-config
```

## 🚨 AlertManager 설정

### 알림 규칙 정의

```yaml
# custom-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: custom-alerts
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
    role: alert-rules
spec:
  groups:
  - name: kubernetes-cluster
    rules:
    - alert: NodeDown
      expr: up{job="node-exporter"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Node {{ $labels.instance }} is down"
        description: "Node {{ $labels.instance }} has been down for more than 5 minutes."
        
    - alert: HighCPUUsage
      expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage on {{ $labels.instance }}"
        description: "CPU usage is above 80% for more than 10 minutes."
        
    - alert: HighMemoryUsage
      expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage on {{ $labels.instance }}"
        description: "Memory usage is above 90% for more than 5 minutes."
        
    - alert: DiskSpaceLow
      expr: (1 - (node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"})) * 100 > 85
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Low disk space on {{ $labels.instance }}"
        description: "Disk usage is above 85% on filesystem {{ $labels.mountpoint }}."
        
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has been restarting frequently."
        
  - name: etcd
    rules:
    - alert: etcdBackendCommitDurationHigh
      expr: histogram_quantile(0.99, sum(rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])) by (instance, le)) > 0.25
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "etcd backend commit duration is high"
        description: "99th percentile of etcd disk backend commit duration is {{ $value }}s on {{ $labels.instance }}."
```

### AlertManager 설정

```yaml
# alertmanager-config.yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-prometheus-stack-kube-prom-alertmanager
  namespace: monitoring
type: Opaque
stringData:
  alertmanager.yml: |
    global:
      smtp_smarthost: 'mail.company.local:587'
      smtp_from: 'alerts@company.local'
      smtp_auth_username: 'alerts@company.local'
      smtp_auth_password: 'password123'
      
    templates:
    - '/etc/alertmanager/templates/*.tmpl'
    
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 10s
      group_interval: 10m
      repeat_interval: 12h
      receiver: 'web.hook'
      routes:
      - match:
          severity: critical
        receiver: critical-alerts
        continue: true
      - match:
          severity: warning
        receiver: warning-alerts
        
    receivers:
    - name: 'web.hook'
      webhook_configs:
      - url: 'http://webhook.company.local/alerts'
        
    - name: 'critical-alerts'
      email_configs:
      - to: 'oncall@company.local'
        subject: '[CRITICAL] {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          Labels: {{ range .Labels.SortedPairs }}{{ .Name }}={{ .Value }} {{ end }}
          {{ end }}
      slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
        channel: '#alerts-critical'
        title: 'Critical Alert'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
        
    - name: 'warning-alerts'
      email_configs:
      - to: 'team@company.local'
        subject: '[WARNING] {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          {{ end }}
```

## 📈 Grafana 대시보드 커스터마이징

### 온프렘 클러스터 대시보드

```json
{
  "dashboard": {
    "id": null,
    "title": "OnPrem Kubernetes Cluster Overview",
    "tags": ["kubernetes", "onprem"],
    "timezone": "Asia/Seoul",
    "panels": [
      {
        "id": 1,
        "title": "Cluster Nodes Status",
        "type": "stat",
        "targets": [
          {
            "expr": "kube_node_status_condition{condition=\"Ready\",status=\"true\"}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "color": {
              "mode": "thresholds"
            },
            "thresholds": {
              "steps": [
                {"color": "red", "value": 0},
                {"color": "green", "value": 1}
              ]
            }
          }
        }
      },
      {
        "id": 2,
        "title": "CPU Usage by Node",
        "type": "timeseries",
        "targets": [
          {
            "expr": "100 - (avg by(instance) (irate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)"
          }
        ]
      },
      {
        "id": 3,
        "title": "Memory Usage by Node",
        "type": "timeseries", 
        "targets": [
          {
            "expr": "(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100"
          }
        ]
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "30s"
  }
}
```

## 🔍 모니터링 검증

### 메트릭 수집 확인

```bash
# Prometheus 타겟 상태 확인
kubectl port-forward -n monitoring svc/prometheus-stack-kube-prom-prometheus 9090:9090

# 브라우저에서 http://localhost:9090/targets 접속하여 확인

# 주요 메트릭 쿼리 테스트
curl -G 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=up{job="node-exporter"}'

curl -G 'http://localhost:9090/api/v1/query' \
  --data-urlencode 'query=kube_node_status_condition{condition="Ready"}'
```

### Grafana 접속 및 설정

```bash
# Grafana 접속
kubectl port-forward -n monitoring svc/prometheus-stack-grafana 3000:80

# 기본 로그인: admin / admin123!@#
# http://localhost:3000 접속

# Prometheus 데이터소스 확인
# Configuration > Data Sources > Prometheus
# URL: http://prometheus-stack-kube-prom-prometheus:9090
```

## 🛠️ 성능 튜닝

### Prometheus 최적화

```yaml
# prometheus-performance.yaml
prometheus:
  prometheusSpec:
    # 메모리 사용량 최적화
    resources:
      requests:
        memory: 4Gi
        cpu: 1000m
      limits:
        memory: 8Gi
        cpu: 2000m
    
    # 압축 및 보존 정책
    retention: 15d
    retentionSize: 40GB
    
    # WAL 압축
    walCompression: true
    
    # 스크래핑 최적화
    scrapeInterval: 30s
    scrapeTimeout: 10s
    
    # 쿼리 타임아웃
    queryTimeout: 30s
    
    # 동시 쿼리 제한
    queryConcurrency: 20
    
    # 메모리 제한
    queryMaxConcurrency: 20
    queryMaxSamples: 50000000
```

### 장기 스토리지 설정 (Thanos)

```yaml
# thanos-sidecar.yaml
prometheus:
  prometheusSpec:
    thanos:
      image: quay.io/thanos/thanos:v0.32.2
      version: v0.32.2
      objectStorageConfig:
        key: thanos.yaml
        name: thanos-objstore-secret
```

## 🚨 실전 운영 팁

### 🔧 메트릭 최적화
1. **불필요한 메트릭 제거**: `metric_relabel_configs` 활용
2. **샘플링 간격 조정**: 중요도에 따라 간격 차별화
3. **메모리 관리**: Prometheus 메모리 사용량 모니터링

### 📊 대시보드 관리
1. **표준화**: 팀 내 대시보드 템플릿 정의
2. **성능 고려**: 무거운 쿼리 최소화
3. **사용자별 접근**: RBAC 적용

### 🚨 알림 정책
1. **알림 피로 방지**: 적절한 임계값과 지연 시간 설정
2. **에스컬레이션**: 심각도별 알림 대상 분리
3. **침묵 규칙**: 점검 시간 알림 차단

---

> 💡 **실전 경험**: 온프렘 환경에서는 스토리지 용량 관리가 중요합니다. 메트릭 보존 기간과 해상도를 신중히 설정하고, 정기적인 용량 모니터링을 수행하세요.

태그: #monitoring #prometheus #grafana #alertmanager #onprem