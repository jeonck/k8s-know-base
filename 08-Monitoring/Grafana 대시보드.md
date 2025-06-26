# Grafana 대시보드

## 🎯 개요

쿠버네티스 클러스터 모니터링을 위한 Grafana 대시보드 구성, 실전 대시보드 템플릿, 알림 설정, 그리고 온프렘 환경에 최적화된 시각화 전략을 다룹니다.

## 📊 Grafana 설치와 기본 설정

### Grafana 배포

```yaml
# grafana-deployment.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-config
  namespace: monitoring
data:
  grafana.ini: |
    [analytics]
    check_for_updates = true
    
    [grafana_net]
    url = https://grafana.net
    
    [log]
    mode = console
    
    [paths]
    data = /var/lib/grafana/data
    logs = /var/log/grafana
    plugins = /var/lib/grafana/plugins
    provisioning = /etc/grafana/provisioning
    
    [security]
    admin_user = admin
    admin_password = $__env{GF_SECURITY_ADMIN_PASSWORD}
    
    [server]
    http_port = 3000
    root_url = http://grafana.monitoring.local
    
    [users]
    allow_sign_up = false
    auto_assign_org = true
    auto_assign_org_role = Viewer
    
    [auth.anonymous]
    enabled = false
    
    [auth.ldap]
    enabled = false
    config_file = /etc/grafana/ldap.toml

---
apiVersion: v1
kind: Secret
metadata:
  name: grafana-secret
  namespace: monitoring
type: Opaque
data:
  admin-password: YWRtaW5wYXNzd29yZA==  # adminpassword (base64)

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 472
        fsGroup: 472
      containers:
      - name: grafana
        image: grafana/grafana:9.3.2
        ports:
        - containerPort: 3000
          name: http-grafana
          protocol: TCP
        env:
        - name: GF_SECURITY_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: grafana-secret
              key: admin-password
        - name: GF_INSTALL_PLUGINS
          value: "grafana-piechart-panel,grafana-worldmap-panel,grafana-clock-panel"
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /robots.txt
            port: 3000
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 30
          successThreshold: 1
          timeoutSeconds: 2
        livenessProbe:
          failureThreshold: 3
          initialDelaySeconds: 30
          periodSeconds: 10
          successThreshold: 1
          tcpSocket:
            port: 3000
          timeoutSeconds: 1
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        volumeMounts:
        - name: grafana-config
          mountPath: /etc/grafana/grafana.ini
          subPath: grafana.ini
        - name: grafana-data
          mountPath: /var/lib/grafana
        - name: grafana-dashboards
          mountPath: /etc/grafana/provisioning/dashboards
        - name: grafana-datasources
          mountPath: /etc/grafana/provisioning/datasources
      volumes:
      - name: grafana-config
        configMap:
          name: grafana-config
      - name: grafana-data
        persistentVolumeClaim:
          claimName: grafana-pvc
      - name: grafana-dashboards
        configMap:
          name: grafana-dashboard-provisioning
      - name: grafana-datasources
        configMap:
          name: grafana-datasource-provisioning

---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  selector:
    app: grafana
  ports:
  - port: 3000
    targetPort: http-grafana
    name: http
  type: ClusterIP

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: grafana-storage
```

### 데이터소스 설정

```yaml
# grafana-datasources.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasource-provisioning
  namespace: monitoring
data:
  datasource.yaml: |
    apiVersion: 1
    
    deleteDatasources:
      - name: Prometheus
        orgId: 1
    
    datasources:
      - name: Prometheus
        type: prometheus
        access: proxy
        orgId: 1
        url: http://prometheus:9090
        basicAuth: false
        isDefault: true
        version: 1
        editable: true
        jsonData:
          httpMethod: POST
          timeInterval: "30s"
          
      - name: AlertManager
        type: alertmanager
        access: proxy
        orgId: 1
        url: http://alertmanager:9093
        basicAuth: false
        version: 1
        editable: true
        
      - name: Loki
        type: loki
        access: proxy
        orgId: 1
        url: http://loki:3100
        basicAuth: false
        version: 1
        editable: true
        jsonData:
          maxLines: 1000
          
      - name: Jaeger
        type: jaeger
        access: proxy
        orgId: 1
        url: http://jaeger-query:16686
        basicAuth: false
        version: 1
        editable: true
```

### 대시보드 프로비저닝

```yaml
# grafana-dashboard-provisioning.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-provisioning
  namespace: monitoring
data:
  dashboard.yaml: |
    apiVersion: 1
    
    providers:
      - name: 'default'
        orgId: 1
        folder: ''
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards
          
      - name: 'kubernetes'
        orgId: 1
        folder: 'Kubernetes'
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/kubernetes
          
      - name: 'applications'
        orgId: 1
        folder: 'Applications'
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/applications
```

## 📈 실전 대시보드 템플릿

### 클러스터 개요 대시보드

```json
{
  "dashboard": {
    "id": null,
    "title": "Kubernetes Cluster Overview",
    "tags": ["kubernetes", "cluster"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Cluster Health",
        "type": "stat",
        "targets": [
          {
            "expr": "up{job=\"kubernetes-apiservers\"}",
            "legendFormat": "API Server",
            "refId": "A"
          },
          {
            "expr": "up{job=\"kubernetes-nodes\"}",
            "legendFormat": "Nodes",
            "refId": "B"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "color": {
              "mode": "thresholds"
            },
            "thresholds": {
              "steps": [
                {
                  "color": "red",
                  "value": 0
                },
                {
                  "color": "green",
                  "value": 1
                }
              ]
            }
          }
        },
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 0,
          "y": 0
        }
      },
      {
        "id": 2,
        "title": "Node Resources",
        "type": "graph",
        "targets": [
          {
            "expr": "(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100",
            "legendFormat": "Memory Usage % - {{instance}}",
            "refId": "A"
          },
          {
            "expr": "100 - (avg by (instance) (irate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
            "legendFormat": "CPU Usage % - {{instance}}",
            "refId": "B"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 12,
          "y": 0
        }
      },
      {
        "id": 3,
        "title": "Pod Status",
        "type": "piechart",
        "targets": [
          {
            "expr": "sum by (phase) (kube_pod_status_phase)",
            "legendFormat": "{{phase}}",
            "refId": "A"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 8,
          "x": 0,
          "y": 8
        }
      },
      {
        "id": 4,
        "title": "Network I/O",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(node_network_receive_bytes_total[5m])",
            "legendFormat": "Receive - {{instance}} - {{device}}",
            "refId": "A"
          },
          {
            "expr": "rate(node_network_transmit_bytes_total[5m])",
            "legendFormat": "Transmit - {{instance}} - {{device}}",
            "refId": "B"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 16,
          "x": 8,
          "y": 8
        }
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "5s"
  }
}
```

### 노드 상세 대시보드

```bash
#!/bin/bash
# create-node-dashboard.sh

cat > node-dashboard.json << 'EOF'
{
  "dashboard": {
    "id": null,
    "title": "Kubernetes Node Details",
    "tags": ["kubernetes", "node"],
    "templating": {
      "list": [
        {
          "name": "node",
          "type": "query",
          "query": "label_values(kube_node_info, node)",
          "refresh": 1,
          "includeAll": false,
          "multi": false
        }
      ]
    },
    "panels": [
      {
        "id": 1,
        "title": "CPU Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "100 - (avg by (instance) (irate(node_cpu_seconds_total{mode=\"idle\", instance=~\"$node:.*\"}[5m])) * 100)",
            "legendFormat": "CPU Usage",
            "refId": "A"
          }
        ],
        "yAxes": [
          {
            "max": 100,
            "min": 0,
            "unit": "percent"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 0,
          "y": 0
        }
      },
      {
        "id": 2,
        "title": "Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "(1 - (node_memory_MemAvailable_bytes{instance=~\"$node:.*\"} / node_memory_MemTotal_bytes{instance=~\"$node:.*\"})) * 100",
            "legendFormat": "Memory Usage",
            "refId": "A"
          }
        ],
        "yAxes": [
          {
            "max": 100,
            "min": 0,
            "unit": "percent"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 12,
          "y": 0
        }
      },
      {
        "id": 3,
        "title": "Disk Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "(1 - (node_filesystem_avail_bytes{instance=~\"$node:.*\", fstype!=\"tmpfs\"} / node_filesystem_size_bytes{instance=~\"$node:.*\", fstype!=\"tmpfs\"})) * 100",
            "legendFormat": "{{mountpoint}}",
            "refId": "A"
          }
        ],
        "yAxes": [
          {
            "max": 100,
            "min": 0,
            "unit": "percent"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 0,
          "y": 8
        }
      },
      {
        "id": 4,
        "title": "Network Traffic",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(node_network_receive_bytes_total{instance=~\"$node:.*\", device!=\"lo\"}[5m])",
            "legendFormat": "Receive - {{device}}",
            "refId": "A"
          },
          {
            "expr": "rate(node_network_transmit_bytes_total{instance=~\"$node:.*\", device!=\"lo\"}[5m])",
            "legendFormat": "Transmit - {{device}}",
            "refId": "B"
          }
        ],
        "yAxes": [
          {
            "unit": "Bps"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 12,
          "y": 8
        }
      },
      {
        "id": 5,
        "title": "Pod Count",
        "type": "stat",
        "targets": [
          {
            "expr": "count(kube_pod_info{node=\"$node\"})",
            "legendFormat": "Running Pods",
            "refId": "A"
          }
        ],
        "gridPos": {
          "h": 4,
          "w": 6,
          "x": 0,
          "y": 16
        }
      },
      {
        "id": 6,
        "title": "Load Average",
        "type": "graph",
        "targets": [
          {
            "expr": "node_load1{instance=~\"$node:.*\"}",
            "legendFormat": "1m",
            "refId": "A"
          },
          {
            "expr": "node_load5{instance=~\"$node:.*\"}",
            "legendFormat": "5m",
            "refId": "B"
          },
          {
            "expr": "node_load15{instance=~\"$node:.*\"}",
            "legendFormat": "15m",
            "refId": "C"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 18,
          "x": 6,
          "y": 16
        }
      }
    ]
  }
}
EOF

echo "Node dashboard template created: node-dashboard.json"
```

### 애플리케이션 모니터링 대시보드

```bash
#!/bin/bash
# create-application-dashboard.sh

cat > application-dashboard.json << 'EOF'
{
  "dashboard": {
    "id": null,
    "title": "Application Monitoring",
    "tags": ["kubernetes", "application"],
    "templating": {
      "list": [
        {
          "name": "namespace",
          "type": "query",
          "query": "label_values(kube_namespace_labels, namespace)",
          "refresh": 1,
          "includeAll": true,
          "multi": true
        },
        {
          "name": "deployment",
          "type": "query",
          "query": "label_values(kube_deployment_labels{namespace=~\"$namespace\"}, deployment)",
          "refresh": 1,
          "includeAll": true,
          "multi": true
        }
      ]
    },
    "panels": [
      {
        "id": 1,
        "title": "Pod Restart Count",
        "type": "graph",
        "targets": [
          {
            "expr": "increase(kube_pod_container_status_restarts_total{namespace=~\"$namespace\", pod=~\"$deployment.*\"}[5m])",
            "legendFormat": "{{pod}} - {{container}}",
            "refId": "A"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 0,
          "y": 0
        }
      },
      {
        "id": 2,
        "title": "CPU Usage by Pod",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(container_cpu_usage_seconds_total{namespace=~\"$namespace\", pod=~\"$deployment.*\", container!=\"POD\"}[5m])",
            "legendFormat": "{{pod}} - {{container}}",
            "refId": "A"
          }
        ],
        "yAxes": [
          {
            "unit": "cores"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 12,
          "y": 0
        }
      },
      {
        "id": 3,
        "title": "Memory Usage by Pod",
        "type": "graph",
        "targets": [
          {
            "expr": "container_memory_working_set_bytes{namespace=~\"$namespace\", pod=~\"$deployment.*\", container!=\"POD\"}",
            "legendFormat": "{{pod}} - {{container}}",
            "refId": "A"
          }
        ],
        "yAxes": [
          {
            "unit": "bytes"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 0,
          "y": 8
        }
      },
      {
        "id": 4,
        "title": "Network I/O by Pod",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(container_network_receive_bytes_total{namespace=~\"$namespace\", pod=~\"$deployment.*\"}[5m])",
            "legendFormat": "Receive - {{pod}}",
            "refId": "A"
          },
          {
            "expr": "rate(container_network_transmit_bytes_total{namespace=~\"$namespace\", pod=~\"$deployment.*\"}[5m])",
            "legendFormat": "Transmit - {{pod}}",
            "refId": "B"
          }
        ],
        "yAxes": [
          {
            "unit": "Bps"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 12,
          "y": 8
        }
      },
      {
        "id": 5,
        "title": "HTTP Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{namespace=~\"$namespace\"}[5m]))",
            "legendFormat": "95th percentile",
            "refId": "A"
          },
          {
            "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket{namespace=~\"$namespace\"}[5m]))",
            "legendFormat": "50th percentile",
            "refId": "B"
          }
        ],
        "yAxes": [
          {
            "unit": "s"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 0,
          "y": 16
        }
      },
      {
        "id": 6,
        "title": "HTTP Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total{namespace=~\"$namespace\"}[5m])",
            "legendFormat": "{{method}} {{status}}",
            "refId": "A"
          }
        ],
        "yAxes": [
          {
            "unit": "reqps"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 12,
          "y": 16
        }
      }
    ]
  }
}
EOF

echo "Application dashboard template created: application-dashboard.json"
```

## 🚨 알림과 통지 설정

### Grafana 알림 규칙

```json
{
  "alert": {
    "id": 1,
    "name": "High CPU Usage",
    "message": "CPU usage is above 80% for more than 5 minutes",
    "frequency": "30s",
    "conditions": [
      {
        "query": {
          "params": ["A", "5m", "now"]
        },
        "reducer": {
          "type": "avg",
          "params": []
        },
        "evaluator": {
          "params": [80],
          "type": "gt"
        }
      }
    ],
    "executionErrorState": "alerting",
    "noDataState": "no_data",
    "for": "5m"
  }
}
```

### 알림 채널 설정

```bash
#!/bin/bash
# setup-notification-channels.sh

# Slack 알림 채널 생성
create_slack_channel() {
    local webhook_url=$1
    local channel_name=${2:-"kubernetes-alerts"}
    
    curl -X POST \
        -H "Authorization: Bearer $GRAFANA_API_KEY" \
        -H "Content-Type: application/json" \
        -d "{
            \"name\": \"slack-alerts\",
            \"type\": \"slack\",
            \"settings\": {
                \"url\": \"$webhook_url\",
                \"username\": \"grafana\",
                \"channel\": \"#$channel_name\",
                \"iconEmoji\": \":exclamation:\",
                \"title\": \"{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}\",
                \"text\": \"{{ range .Alerts }}{{ .Annotations.description }}{{ end }}\"
            }
        }" \
        http://grafana:3000/api/alert-notifications
}

# 이메일 알림 채널 생성
create_email_channel() {
    local email_addresses=$1
    
    curl -X POST \
        -H "Authorization: Bearer $GRAFANA_API_KEY" \
        -H "Content-Type: application/json" \
        -d "{
            \"name\": \"email-alerts\",
            \"type\": \"email\",
            \"settings\": {
                \"addresses\": \"$email_addresses\",
                \"singleEmail\": false
            }
        }" \
        http://grafana:3000/api/alert-notifications
}

# 웹훅 알림 채널 생성
create_webhook_channel() {
    local webhook_url=$1
    
    curl -X POST \
        -H "Authorization: Bearer $GRAFANA_API_KEY" \
        -H "Content-Type: application/json" \
        -d "{
            \"name\": \"webhook-alerts\",
            \"type\": \"webhook\",
            \"settings\": {
                \"url\": \"$webhook_url\",
                \"httpMethod\": \"POST\",
                \"maxAlerts\": 0
            }
        }" \
        http://grafana:3000/api/alert-notifications
}

# 사용 예제
# create_slack_channel "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK" "kubernetes-alerts"
# create_email_channel "admin@company.com,ops@company.com"
# create_webhook_channel "http://alert-manager:9093/api/v1/alerts"
```

## 📱 모바일 대시보드 최적화

### 모바일 친화적 대시보드

```json
{
  "dashboard": {
    "id": null,
    "title": "Mobile Kubernetes Overview",
    "tags": ["kubernetes", "mobile"],
    "panels": [
      {
        "id": 1,
        "title": "Cluster Status",
        "type": "stat",
        "targets": [
          {
            "expr": "up{job=\"kubernetes-apiservers\"}",
            "legendFormat": "API Server"
          }
        ],
        "gridPos": {
          "h": 4,
          "w": 24,
          "x": 0,
          "y": 0
        },
        "options": {
          "colorMode": "background",
          "graphMode": "none",
          "justifyMode": "center",
          "orientation": "horizontal"
        }
      },
      {
        "id": 2,
        "title": "Critical Alerts",
        "type": "table",
        "targets": [
          {
            "expr": "ALERTS{alertstate=\"firing\", severity=\"critical\"}",
            "format": "table"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 24,
          "x": 0,
          "y": 4
        }
      },
      {
        "id": 3,
        "title": "Resource Usage",
        "type": "bargauge",
        "targets": [
          {
            "expr": "avg(100 - (avg by (instance) (irate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100))",
            "legendFormat": "CPU %"
          },
          {
            "expr": "avg((1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100)",
            "legendFormat": "Memory %"
          }
        ],
        "gridPos": {
          "h": 6,
          "w": 24,
          "x": 0,
          "y": 12
        },
        "options": {
          "orientation": "horizontal",
          "displayMode": "gradient"
        }
      }
    ]
  }
}
```

## 🔧 대시보드 관리 자동화

### 대시보드 백업 및 복원

```bash
#!/bin/bash
# dashboard-backup.sh

GRAFANA_URL="http://grafana:3000"
GRAFANA_API_KEY="your-api-key"
BACKUP_DIR="/backups/grafana-dashboards"

# 대시보드 백업
backup_dashboards() {
    echo "Starting dashboard backup..."
    
    mkdir -p "$BACKUP_DIR"
    timestamp=$(date +%Y%m%d_%H%M%S)
    backup_file="$BACKUP_DIR/dashboards-backup-$timestamp.tar.gz"
    
    # 모든 대시보드 목록 가져오기
    dashboards=$(curl -s -H "Authorization: Bearer $GRAFANA_API_KEY" \
        "$GRAFANA_URL/api/search?type=dash-db" | \
        jq -r '.[].uid')
    
    # 임시 디렉토리 생성
    temp_dir=$(mktemp -d)
    
    # 각 대시보드 백업
    for uid in $dashboards; do
        echo "Backing up dashboard: $uid"
        curl -s -H "Authorization: Bearer $GRAFANA_API_KEY" \
            "$GRAFANA_URL/api/dashboards/uid/$uid" | \
            jq '.dashboard' > "$temp_dir/${uid}.json"
    done
    
    # 압축
    tar -czf "$backup_file" -C "$temp_dir" .
    rm -rf "$temp_dir"
    
    echo "Backup completed: $backup_file"
}

# 대시보드 복원
restore_dashboards() {
    local backup_file=$1
    
    if [ ! -f "$backup_file" ]; then
        echo "Backup file not found: $backup_file"
        exit 1
    fi
    
    echo "Restoring dashboards from: $backup_file"
    
    # 임시 디렉토리에 압축 해제
    temp_dir=$(mktemp -d)
    tar -xzf "$backup_file" -C "$temp_dir"
    
    # 각 대시보드 복원
    for dashboard_file in "$temp_dir"/*.json; do
        if [ -f "$dashboard_file" ]; then
            echo "Restoring dashboard: $(basename $dashboard_file)"
            
            # 대시보드 JSON 준비
            dashboard_json=$(cat "$dashboard_file")
            
            # Grafana에 업로드
            curl -X POST \
                -H "Authorization: Bearer $GRAFANA_API_KEY" \
                -H "Content-Type: application/json" \
                -d "{
                    \"dashboard\": $dashboard_json,
                    \"overwrite\": true
                }" \
                "$GRAFANA_URL/api/dashboards/db"
        fi
    done
    
    rm -rf "$temp_dir"
    echo "Restore completed"
}

# 대시보드 동기화 (GitOps)
sync_dashboards() {
    local git_repo=$1
    local dashboard_dir="$git_repo/grafana-dashboards"
    
    echo "Syncing dashboards from Git repository..."
    
    # Git 저장소 클론 또는 업데이트
    if [ -d "$git_repo" ]; then
        cd "$git_repo" && git pull
    else
        git clone "$git_repo" /tmp/dashboard-repo
        git_repo="/tmp/dashboard-repo"
        dashboard_dir="$git_repo/grafana-dashboards"
    fi
    
    # 대시보드 파일들 업로드
    for dashboard_file in "$dashboard_dir"/*.json; do
        if [ -f "$dashboard_file" ]; then
            echo "Syncing dashboard: $(basename $dashboard_file)"
            
            dashboard_json=$(cat "$dashboard_file")
            
            curl -X POST \
                -H "Authorization: Bearer $GRAFANA_API_KEY" \
                -H "Content-Type: application/json" \
                -d "{
                    \"dashboard\": $dashboard_json,
                    \"overwrite\": true
                }" \
                "$GRAFANA_URL/api/dashboards/db"
        fi
    done
}

case "${1:-backup}" in
    backup)
        backup_dashboards
        ;;
    restore)
        restore_dashboards "$2"
        ;;
    sync)
        sync_dashboards "$2"
        ;;
    *)
        echo "Usage: $0 {backup|restore <backup-file>|sync <git-repo>}"
        exit 1
        ;;
esac
```

### 대시보드 템플릿 생성기

```bash
#!/bin/bash
# dashboard-generator.sh

generate_node_dashboard() {
    local node_name=$1
    
    cat > "node-${node_name}-dashboard.json" << EOF
{
  "dashboard": {
    "title": "Node: $node_name",
    "tags": ["kubernetes", "node", "$node_name"],
    "templating": {
      "list": [
        {
          "name": "node",
          "type": "constant",
          "current": {
            "value": "$node_name"
          },
          "hide": 2
        }
      ]
    },
    "panels": [
      {
        "title": "CPU Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "100 - (avg by (instance) (irate(node_cpu_seconds_total{mode=\"idle\", instance=~\"$node:.*\"}[5m])) * 100)"
          }
        ]
      }
    ]
  }
}
EOF
    
    echo "Generated dashboard for node: $node_name"
}

generate_namespace_dashboard() {
    local namespace=$1
    
    cat > "namespace-${namespace}-dashboard.json" << EOF
{
  "dashboard": {
    "title": "Namespace: $namespace",
    "tags": ["kubernetes", "namespace", "$namespace"],
    "templating": {
      "list": [
        {
          "name": "namespace",
          "type": "constant",
          "current": {
            "value": "$namespace"
          },
          "hide": 2
        }
      ]
    },
    "panels": [
      {
        "title": "Pod Count",
        "type": "stat",
        "targets": [
          {
            "expr": "count(kube_pod_info{namespace=\"$namespace\"})"
          }
        ]
      }
    ]
  }
}
EOF
    
    echo "Generated dashboard for namespace: $namespace"
}

# 모든 노드에 대한 대시보드 생성
generate_all_node_dashboards() {
    kubectl get nodes -o name | cut -d/ -f2 | while read node; do
        generate_node_dashboard "$node"
    done
}

# 모든 네임스페이스에 대한 대시보드 생성
generate_all_namespace_dashboards() {
    kubectl get namespaces -o name | cut -d/ -f2 | while read ns; do
        generate_namespace_dashboard "$ns"
    done
}

case "${1:-help}" in
    node)
        generate_node_dashboard "$2"
        ;;
    namespace)
        generate_namespace_dashboard "$2"
        ;;
    all-nodes)
        generate_all_node_dashboards
        ;;
    all-namespaces)
        generate_all_namespace_dashboards
        ;;
    *)
        echo "Usage: $0 {node <node-name>|namespace <namespace>|all-nodes|all-namespaces}"
        exit 1
        ;;
esac
```

## 📋 대시보드 베스트 프랙티스

### 1. 대시보드 설계 원칙
- **계층적 구조**: 개요 → 상세 → 드릴다운
- **사용자별 맞춤**: 역할에 따른 대시보드 분리
- **성능 최적화**: 쿼리 최적화와 적절한 시간 범위
- **일관성**: 색상, 단위, 명명 규칙 통일

### 2. 필수 대시보드 목록
- [ ] 클러스터 개요 대시보드
- [ ] 노드 상세 모니터링
- [ ] 네임스페이스별 리소스 사용량
- [ ] 애플리케이션 성능 모니터링
- [ ] 네트워크 트래픽 분석
- [ ] 스토리지 사용량 추적

### 3. 알림 설정 가이드
- **임계값 설정**: 환경에 맞는 적절한 임계값
- **알림 피로도 방지**: 중요도에 따른 알림 분류
- **에스컬레이션**: 단계별 알림 전달 체계

---

> 💡 **실전 경험**: Grafana 대시보드는 처음에는 간단하게 시작해서 점진적으로 개선하는 것이 좋습니다. 너무 많은 패널을 한 번에 추가하면 성능 저하와 복잡성 증가를 야기할 수 있습니다. 특히 온프렘 환경에서는 네트워크 지연을 고려한 쿼리 최적화가 중요합니다.

태그: #grafana #dashboard #monitoring #visualization #onprem
