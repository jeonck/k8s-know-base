# Pod 설계 패턴

## 🎯 개요

Pod는 쿠버네티스의 최소 배포 단위입니다. 효율적이고 안정적인 애플리케이션 운영을 위한 Pod 설계 패턴과 베스트 프랙티스를 다룹니다.

## 📦 기본 Pod 설계 원칙

### Single Responsibility Principle
```yaml
# good-pod-design.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-server
  labels:
    app: web-server
    version: "1.0"
    tier: frontend
spec:
  containers:
  - name: web-server
    image: nginx:1.21-alpine
    ports:
    - containerPort: 80
      name: http
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
    # 헬스체크 설정
    livenessProbe:
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 30
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
    # 환경 변수
    env:
    - name: LOG_LEVEL
      value: "info"
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database.host
```

### 멀티 컨테이너 Pod 패턴

#### 1. Sidecar 패턴
```yaml
# sidecar-pattern.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
  # 메인 애플리케이션
  - name: main-app
    image: myapp:1.0
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app
    
  # 로그 수집 사이드카
  - name: log-collector
    image: fluentd:v1.14
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app
      readOnly: true
    - name: fluentd-config
      mountPath: /fluentd/etc
    env:
    - name: FLUENTD_CONF
      value: "fluent.conf"
      
  volumes:
  - name: shared-logs
    emptyDir: {}
  - name: fluentd-config
    configMap:
      name: fluentd-config

---
# 모니터링 사이드카 예제
apiVersion: v1
kind: Pod
metadata:
  name: app-with-monitoring
spec:
  containers:
  # 메인 애플리케이션
  - name: web-app
    image: mywebapp:1.0
    ports:
    - containerPort: 8080
    
  # Prometheus 메트릭 익스포터
  - name: metrics-exporter
    image: prom/node-exporter:latest
    ports:
    - containerPort: 9100
    args:
    - --path.procfs=/host/proc
    - --path.sysfs=/host/sys
    - --collector.filesystem.ignored-mount-points
    - "^/(sys|proc|dev|host|etc|rootfs/var/lib/docker/containers|rootfs/var/lib/docker/overlay2|rootfs/run/docker/netns|rootfs/var/lib/docker/aufs)($$|/)"
    volumeMounts:
    - name: proc
      mountPath: /host/proc
      readOnly: true
    - name: sys
      mountPath: /host/sys
      readOnly: true
      
  volumes:
  - name: proc
    hostPath:
      path: /proc
  - name: sys
    hostPath:
      path: /sys
```

#### 2. Ambassador 패턴
```yaml
# ambassador-pattern.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-ambassador
spec:
  containers:
  # 메인 애플리케이션
  - name: main-app
    image: myapp:1.0
    env:
    - name: DATABASE_URL
      value: "localhost:3306"  # Ambassador를 통해 접근
    
  # Database 프록시 Ambassador
  - name: db-ambassador
    image: envoyproxy/envoy:v1.24.0
    ports:
    - containerPort: 3306
    volumeMounts:
    - name: envoy-config
      mountPath: /etc/envoy
    command: ["/usr/local/bin/envoy"]
    args: ["-c", "/etc/envoy/envoy.yaml"]
    
  volumes:
  - name: envoy-config
    configMap:
      name: envoy-proxy-config

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-proxy-config
data:
  envoy.yaml: |
    static_resources:
      listeners:
      - name: database_listener
        address:
          socket_address:
            address: 0.0.0.0
            port_value: 3306
        filter_chains:
        - filters:
          - name: envoy.filters.network.tcp_proxy
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
              stat_prefix: database_tcp
              cluster: database_cluster
      clusters:
      - name: database_cluster
        connect_timeout: 0.25s
        type: STRICT_DNS
        lb_policy: ROUND_ROBIN
        load_assignment:
          cluster_name: database_cluster
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: mysql-primary.database.svc.cluster.local
                    port_value: 3306
            - endpoint:
                address:
                  socket_address:
                    address: mysql-replica.database.svc.cluster.local
                    port_value: 3306
```

#### 3. Adapter 패턴
```yaml
# adapter-pattern.yaml
apiVersion: v1
kind: Pod
metadata:
  name: legacy-app-with-adapter
spec:
  containers:
  # 레거시 애플리케이션
  - name: legacy-app
    image: legacy-app:1.0
    volumeMounts:
    - name: shared-data
      mountPath: /data
    
  # 메트릭 어댑터
  - name: metrics-adapter
    image: metrics-adapter:1.0
    ports:
    - containerPort: 9090
    volumeMounts:
    - name: shared-data
      mountPath: /data
      readOnly: true
    env:
    - name: LEGACY_LOG_PATH
      value: "/data/app.log"
    - name: METRICS_FORMAT
      value: "prometheus"
      
  volumes:
  - name: shared-data
    emptyDir: {}
```

## 🔧 리소스 관리 패턴

### QoS (Quality of Service) 클래스

```yaml
# qos-examples.yaml
---
# Guaranteed QoS - 최고 우선순위
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "200Mi"
        cpu: "200m"
      limits:
        memory: "200Mi"  # requests와 동일
        cpu: "200m"      # requests와 동일

---
# Burstable QoS - 중간 우선순위
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "100Mi"
        cpu: "100m"
      limits:
        memory: "200Mi"  # requests보다 높음
        cpu: "200m"

---
# BestEffort QoS - 낮은 우선순위
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
  - name: app
    image: nginx
    # resources 미정의
```

### 리소스 최적화 패턴

```yaml
# resource-optimization.yaml
apiVersion: v1
kind: Pod
metadata:
  name: optimized-pod
spec:
  containers:
  - name: web-server
    image: nginx:alpine
    resources:
      # CPU 버스팅 허용하되 메모리는 엄격히 제한
      requests:
        memory: "128Mi"
        cpu: "50m"
      limits:
        memory: "128Mi"   # OOMKilled 방지를 위해 requests와 동일
        cpu: "500m"       # 필요시 버스팅 허용
    
    # 성능 모니터링을 위한 메트릭 포트
    ports:
    - containerPort: 80
      name: http
    - containerPort: 9113
      name: metrics
      
    # 효율적인 헬스체크
    livenessProbe:
      httpGet:
        path: /nginx_status
        port: 80
      initialDelaySeconds: 30
      periodSeconds: 30      # 빈번하지 않게
      timeoutSeconds: 5
      failureThreshold: 3
      
    readinessProbe:
      httpGet:
        path: /nginx_status
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
      timeoutSeconds: 3
      successThreshold: 1
      failureThreshold: 3
```

## 🔒 보안 패턴

### Security Context 설정

```yaml
# security-patterns.yaml
---
# 강화된 보안 설정
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  # Pod 레벨 보안 컨텍스트
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
    supplementalGroups: [4000]
    
  containers:
  - name: secure-app
    image: myapp:1.0
    # 컨테이너 레벨 보안 컨텍스트
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE  # 필요한 capability만 추가
        
    # 읽기 전용 루트 파일시스템 사용시 임시 볼륨 마운트
    volumeMounts:
    - name: tmp-volume
      mountPath: /tmp
    - name: var-cache
      mountPath: /var/cache/nginx
    - name: var-run
      mountPath: /var/run
      
  volumes:
  - name: tmp-volume
    emptyDir: {}
  - name: var-cache
    emptyDir: {}
  - name: var-run
    emptyDir: {}

---
# 특권 컨테이너가 필요한 경우 (최소한으로 사용)
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: kube-system
spec:
  hostNetwork: true
  hostPID: true
  containers:
  - name: system-monitor
    image: system-monitor:1.0
    securityContext:
      privileged: true
    volumeMounts:
    - name: host-proc
      mountPath: /host/proc
      readOnly: true
    - name: host-sys
      mountPath: /host/sys
      readOnly: true
      
  volumes:
  - name: host-proc
    hostPath:
      path: /proc
  - name: host-sys
    hostPath:
      path: /sys
```

### 시크릿 관리 패턴

```yaml
# secret-management.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secrets
spec:
  containers:
  - name: app
    image: myapp:1.0
    
    # 환경 변수로 시크릿 주입 (민감하지 않은 정보용)
    env:
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: db.host
    - name: DATABASE_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
          
    # 파일로 시크릿 마운트 (중요한 정보용)
    volumeMounts:
    - name: tls-certs
      mountPath: /etc/ssl/certs
      readOnly: true
    - name: api-keys
      mountPath: /etc/secrets
      readOnly: true
      
  volumes:
  - name: tls-certs
    secret:
      secretName: tls-certificates
      defaultMode: 0400  # 읽기 전용
  - name: api-keys
    secret:
      secretName: api-credentials
      defaultMode: 0400
```

## 🚀 초기화 패턴

### Init Container 활용

```yaml
# init-container-patterns.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  # Init Container들
  initContainers:
  # 1. 데이터베이스 연결 대기
  - name: wait-for-db
    image: busybox:1.35
    command: ['sh', '-c']
    args:
    - |
      echo "Waiting for database..."
      until nslookup mysql.database.svc.cluster.local; do
        echo "Database not ready, waiting..."
        sleep 2
      done
      echo "Database is ready!"
      
  # 2. 설정 파일 다운로드
  - name: download-config
    image: curlimages/curl:7.85.0
    command: ['sh', '-c']
    args:
    - |
      echo "Downloading configuration..."
      curl -o /shared/app-config.json \
        http://config-server.config.svc.cluster.local/config/app
      echo "Configuration downloaded"
    volumeMounts:
    - name: shared-config
      mountPath: /shared
      
  # 3. 데이터베이스 마이그레이션
  - name: db-migration
    image: migrate/migrate:v4.15.2
    command: ['migrate']
    args:
    - '-path'
    - '/migrations'
    - '-database'
    - 'mysql://user:password@mysql.database.svc.cluster.local:3306/mydb'
    - 'up'
    volumeMounts:
    - name: migration-scripts
      mountPath: /migrations
      
  # 메인 애플리케이션
  containers:
  - name: main-app
    image: myapp:1.0
    volumeMounts:
    - name: shared-config
      mountPath: /etc/config
      readOnly: true
      
  volumes:
  - name: shared-config
    emptyDir: {}
  - name: migration-scripts
    configMap:
      name: db-migration-scripts
```

## 📊 모니터링과 관찰성 패턴

### 구조화된 로깅

```yaml
# logging-patterns.yaml
apiVersion: v1
kind: Pod
metadata:
  name: structured-logging-pod
  annotations:
    # Fluentd/Fluent Bit용 파싱 힌트
    fluentbit.io/parser: "json"
    fluentbit.io/exclude: "false"
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    # 구조화된 로깅 설정
    - name: LOG_FORMAT
      value: "json"
    - name: LOG_LEVEL
      value: "info"
    - name: LOG_OUTPUT
      value: "stdout"
      
    # 로그 볼륨 (필요한 경우)
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app
      
  # 로그 전처리 사이드카
  - name: log-processor
    image: fluent/fluent-bit:2.0
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/app
      readOnly: true
    - name: fluent-bit-config
      mountPath: /fluent-bit/etc
      
  volumes:
  - name: log-volume
    emptyDir: {}
  - name: fluent-bit-config
    configMap:
      name: fluent-bit-config
```

### 메트릭 노출 패턴

```yaml
# metrics-patterns.yaml
apiVersion: v1
kind: Pod
metadata:
  name: metrics-enabled-pod
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
spec:
  containers:
  - name: app
    image: myapp:1.0
    ports:
    - containerPort: 8080
      name: http
    - containerPort: 9090
      name: metrics
      
    # 메트릭 관련 환경 변수
    env:
    - name: METRICS_ENABLED
      value: "true"
    - name: METRICS_PORT
      value: "9090"
    - name: METRICS_PATH
      value: "/metrics"
      
    # 헬스체크와 메트릭 엔드포인트 분리
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
```

## 🔄 라이프사이클 관리 패턴

### Graceful Shutdown

```yaml
# graceful-shutdown.yaml
apiVersion: v1
kind: Pod
metadata:
  name: graceful-shutdown-pod
spec:
  terminationGracePeriodSeconds: 60  # 기본 30초에서 연장
  
  containers:
  - name: web-server
    image: nginx:alpine
    
    # 종료 시그널 처리
    lifecycle:
      preStop:
        exec:
          command:
          - /bin/sh
          - -c
          - |
            echo "Graceful shutdown initiated"
            # 새로운 연결 차단
            nginx -s quit
            # 진행 중인 요청 완료 대기
            sleep 10
            echo "Graceful shutdown completed"
            
    # SIGTERM 처리를 위한 환경 설정
    env:
    - name: NGINX_SHUTDOWN_TIMEOUT
      value: "50"
```

### 업데이트 패턴

```yaml
# update-patterns.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-update-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 추가로 생성할 수 있는 Pod 수
      maxUnavailable: 0  # 동시에 사용 불가능한 Pod 수
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: myapp:v2.0
        
        # 무중단 배포를 위한 헬스체크
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
          
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
```

## 🛠️ 실전 운영 팁

### 💡 성능 최적화
1. **이미지 최적화**: 멀티 스테이지 빌드, Alpine 기반 이미지 사용
2. **리소스 튜닝**: requests/limits 적절히 설정, QoS 클래스 고려
3. **헬스체크 최적화**: 불필요한 체크 빈도 줄이기

### 🔒 보안 강화
1. **최소 권한 원칙**: 필요한 capability만 부여
2. **읽기 전용 파일시스템**: 가능한 경우 적용
3. **비특권 사용자**: root 사용 금지

### 📊 관찰성 향상
1. **구조화된 로깅**: JSON 형태로 로그 출력
2. **메트릭 노출**: Prometheus 형태로 메트릭 제공
3. **분산 추적**: Jaeger, Zipkin 등 활용

### 🚀 운영 효율성
1. **적절한 레이블링**: 관리와 셀렉터를 위한 일관된 레이블
2. **리소스 네이밍**: 명확하고 일관된 네이밍 규칙
3. **문서화**: Pod 설계 의도와 운영 방법 문서화

---

> 💡 **실전 경험**: Pod 설계에서 가장 중요한 것은 단일 책임 원칙입니다. 하나의 Pod에는 밀접하게 결합된 컨테이너만 배치하고, 리소스 요구사항을 정확히 측정하여 설정하세요.

태그: #pod #design-patterns #containers #best-practices #kubernetes