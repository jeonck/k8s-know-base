# Deployment 전략

## 🎯 개요

쿠버네티스에서 애플리케이션 배포를 위한 다양한 전략과 패턴을 다룹니다. 롤링 업데이트, 블루/그린 배포, 카나리 배포 등의 실전 구현 방법과 온프렘 환경에서의 최적화 방안을 포함합니다.

## 🚀 기본 Deployment 설정

### 표준 Deployment 구성

```yaml
# basic-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
  labels:
    app: web-app
    version: v1.0.0
    environment: production
spec:
  # 복제본 수
  replicas: 3
  
  # 업데이트 전략
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 추가로 생성할 수 있는 Pod 수
      maxUnavailable: 0    # 업데이트 중 사용 불가능한 Pod 수
  
  # Pod 선택자
  selector:
    matchLabels:
      app: web-app
      version: v1.0.0
  
  # Pod 템플릿
  template:
    metadata:
      labels:
        app: web-app
        version: v1.0.0
        environment: production
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      # 배포 전략 설정
      terminationGracePeriodSeconds: 30
      
      # 보안 컨텍스트
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      
      # 이미지 풀 시크릿
      imagePullSecrets:
      - name: registry-secret
      
      containers:
      - name: web-app
        image: registry.company.com/web-app:v1.0.0
        imagePullPolicy: IfNotPresent
        
        # 포트 설정
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        - name: metrics
          containerPort: 9090
          protocol: TCP
        
        # 리소스 설정
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        
        # 환경 변수
        env:
        - name: APP_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: connection-string
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: redis-url
        
        # 볼륨 마운트
        volumeMounts:
        - name: app-config
          mountPath: /etc/app
          readOnly: true
        - name: tmp
          mountPath: /tmp
        
        # 헬스체크
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
          
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2
          
        # 시작 프로브 (느린 시작 애플리케이션용)
        startupProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 30
      
      # 볼륨
      volumes:
      - name: app-config
        configMap:
          name: app-config
      - name: tmp
        emptyDir: {}
      
      # 노드 어피니티
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-type
                operator: In
                values:
                - worker
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web-app
              topologyKey: kubernetes.io/hostname

---
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  app.properties: |
    server.port=8080
    management.port=9090
    logging.level.root=INFO
  redis-url: "redis://redis-service:6379"

---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: production
type: Opaque
stringData:
  connection-string: "postgresql://user:password@db-service:5432/appdb"
```

## 🔄 롤링 업데이트 전략

### 고급 롤링 업데이트 설정

```yaml
# rolling-update-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-update-app
  namespace: production
  annotations:
    deployment.kubernetes.io/revision: "1"
spec:
  replicas: 6
  
  # 롤링 업데이트 세부 설정
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 50%        # 50% 추가 Pod 허용
      maxUnavailable: 25%  # 25%까지 사용 불가 허용
  
  # 진행 상황 데드라인 (10분)
  progressDeadlineSeconds: 600
  
  # 이전 ReplicaSet 보관 수
  revisionHistoryLimit: 5
  
  selector:
    matchLabels:
      app: rolling-update-app
  
  template:
    metadata:
      labels:
        app: rolling-update-app
        version: "{{ .Values.image.tag }}"
    spec:
      terminationGracePeriodSeconds: 60  # Graceful shutdown 시간
      
      containers:
      - name: app
        image: registry.company.com/app:{{ .Values.image.tag }}
        
        # Graceful shutdown 설정
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                # 새로운 요청 받기 중단
                echo "Stopping accepting new requests..."
                kill -TERM 1
                # 기존 요청 처리 대기
                sleep 30
        
        ports:
        - containerPort: 8080
          name: http
        
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        
        # 세밀한 헬스체크 설정
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: http
          initialDelaySeconds: 10
          periodSeconds: 3
          timeoutSeconds: 2
          failureThreshold: 1
          successThreshold: 1
          
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: http
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
```

### 롤링 업데이트 제어 스크립트

```bash
#!/bin/bash
# rolling-update-controller.sh

DEPLOYMENT_NAME="web-app"
NAMESPACE="production"
NEW_IMAGE="registry.company.com/web-app:v1.1.0"

# 롤링 업데이트 시작
start_rolling_update() {
    echo "Starting rolling update for $DEPLOYMENT_NAME..."
    echo "New image: $NEW_IMAGE"
    
    # 현재 이미지 확인
    current_image=$(kubectl get deployment $DEPLOYMENT_NAME -n $NAMESPACE -o jsonpath='{.spec.template.spec.containers[0].image}')
    echo "Current image: $current_image"
    
    # 업데이트 실행
    kubectl set image deployment/$DEPLOYMENT_NAME -n $NAMESPACE app=$NEW_IMAGE
    
    echo "Rolling update initiated..."
}

# 업데이트 진행 상황 모니터링
monitor_rollout() {
    echo "Monitoring rollout progress..."
    
    # 실시간 상태 모니터링
    kubectl rollout status deployment/$DEPLOYMENT_NAME -n $NAMESPACE --timeout=600s
    
    if [ $? -eq 0 ]; then
        echo "✅ Rolling update completed successfully"
        
        # 최종 상태 확인
        kubectl get deployment $DEPLOYMENT_NAME -n $NAMESPACE
        kubectl get pods -n $NAMESPACE -l app=$DEPLOYMENT_NAME
    else
        echo "❌ Rolling update failed or timed out"
        return 1
    fi
}

# 업데이트 일시 정지
pause_rollout() {
    echo "Pausing rollout..."
    kubectl rollout pause deployment/$DEPLOYMENT_NAME -n $NAMESPACE
    echo "Rollout paused. Use 'resume_rollout' to continue."
}

# 업데이트 재개
resume_rollout() {
    echo "Resuming rollout..."
    kubectl rollout resume deployment/$DEPLOYMENT_NAME -n $NAMESPACE
    monitor_rollout
}

# 롤백 실행
rollback_deployment() {
    local revision=${1:-}
    
    echo "Rolling back deployment..."
    
    if [ -n "$revision" ]; then
        kubectl rollout undo deployment/$DEPLOYMENT_NAME -n $NAMESPACE --to-revision=$revision
    else
        kubectl rollout undo deployment/$DEPLOYMENT_NAME -n $NAMESPACE
    fi
    
    # 롤백 상태 모니터링
    kubectl rollout status deployment/$DEPLOYMENT_NAME -n $NAMESPACE --timeout=300s
    
    if [ $? -eq 0 ]; then
        echo "✅ Rollback completed successfully"
    else
        echo "❌ Rollback failed"
        return 1
    fi
}

# 배포 히스토리 확인
check_rollout_history() {
    echo "Deployment rollout history:"
    kubectl rollout history deployment/$DEPLOYMENT_NAME -n $NAMESPACE
}

# 헬스체크 기반 검증
validate_deployment() {
    echo "Validating deployment health..."
    
    # Pod 상태 확인
    ready_pods=$(kubectl get deployment $DEPLOYMENT_NAME -n $NAMESPACE -o jsonpath='{.status.readyReplicas}')
    desired_pods=$(kubectl get deployment $DEPLOYMENT_NAME -n $NAMESPACE -o jsonpath='{.spec.replicas}')
    
    echo "Ready pods: $ready_pods/$desired_pods"
    
    if [ "$ready_pods" = "$desired_pods" ]; then
        echo "✅ All pods are ready"
        
        # 애플리케이션 헬스체크
        echo "Performing application health check..."
        
        # 서비스 엔드포인트 테스트
        service_ip=$(kubectl get service ${DEPLOYMENT_NAME}-service -n $NAMESPACE -o jsonpath='{.spec.clusterIP}')
        if kubectl run health-check-$$ --image=curlimages/curl --rm -it --restart=Never -- curl -f http://$service_ip/health >/dev/null 2>&1; then
            echo "✅ Application health check passed"
            return 0
        else
            echo "❌ Application health check failed"
            return 1
        fi
    else
        echo "❌ Not all pods are ready"
        return 1
    fi
}

# 메인 함수
case "${1:-start}" in
    start)
        start_rolling_update
        monitor_rollout
        validate_deployment
        ;;
    monitor)
        monitor_rollout
        ;;
    pause)
        pause_rollout
        ;;
    resume)
        resume_rollout
        ;;
    rollback)
        rollback_deployment "$2"
        ;;
    history)
        check_rollout_history
        ;;
    validate)
        validate_deployment
        ;;
    *)
        echo "Usage: $0 {start|monitor|pause|resume|rollback [revision]|history|validate}"
        exit 1
        ;;
esac
```

## 🔵🟢 블루/그린 배포

### 블루/그린 배포 구현

```yaml
# blue-green-deployment.yaml
---
# Blue Deployment (현재 운영 중)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
  namespace: production
  labels:
    app: web-app
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
      version: blue
  template:
    metadata:
      labels:
        app: web-app
        version: blue
    spec:
      containers:
      - name: web-app
        image: registry.company.com/web-app:v1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi

---
# Green Deployment (새 버전)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
  namespace: production
  labels:
    app: web-app
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
      version: green
  template:
    metadata:
      labels:
        app: web-app
        version: green
    spec:
      containers:
      - name: web-app
        image: registry.company.com/web-app:v1.1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi

---
# Production Service (블루 또는 그린을 선택)
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  namespace: production
spec:
  selector:
    app: web-app
    version: blue  # 현재는 blue 버전으로 트래픽 라우팅
  ports:
  - port: 80
    targetPort: 8080

---
# 테스트용 그린 서비스
apiVersion: v1
kind: Service
metadata:
  name: web-app-green-service
  namespace: production
spec:
  selector:
    app: web-app
    version: green
  ports:
  - port: 80
    targetPort: 8080
```

### 블루/그린 배포 자동화

```bash
#!/bin/bash
# blue-green-deploy.sh

NAMESPACE="production"
APP_NAME="web-app"
NEW_VERSION=$1
TIMEOUT=300

if [ -z "$NEW_VERSION" ]; then
    echo "Usage: $0 <new-version>"
    echo "Example: $0 v1.1.0"
    exit 1
fi

# 현재 활성 버전 확인
get_active_version() {
    kubectl get service ${APP_NAME}-service -n $NAMESPACE -o jsonpath='{.spec.selector.version}'
}

# 비활성 버전 확인 (blue ↔ green)
get_inactive_version() {
    local active=$(get_active_version)
    if [ "$active" = "blue" ]; then
        echo "green"
    else
        echo "blue"
    fi
}

# 새 버전 배포
deploy_new_version() {
    local inactive_version=$(get_inactive_version)
    local new_image="registry.company.com/${APP_NAME}:${NEW_VERSION}"
    
    echo "Deploying new version to $inactive_version environment..."
    echo "Image: $new_image"
    
    # 비활성 deployment 업데이트
    kubectl set image deployment/app-${inactive_version} \
        -n $NAMESPACE \
        web-app=$new_image
    
    # 배포 완료 대기
    kubectl rollout status deployment/app-${inactive_version} \
        -n $NAMESPACE \
        --timeout=${TIMEOUT}s
    
    if [ $? -ne 0 ]; then
        echo "❌ Deployment failed"
        return 1
    fi
    
    echo "✅ New version deployed to $inactive_version environment"
    return 0
}

# 헬스체크 실행
health_check() {
    local version=$1
    local service_name="${APP_NAME}-${version}-service"
    
    echo "Performing health check on $version environment..."
    
    # 서비스 IP 확인
    local service_ip=$(kubectl get service $service_name -n $NAMESPACE -o jsonpath='{.spec.clusterIP}' 2>/dev/null)
    
    if [ -z "$service_ip" ]; then
        echo "❌ Service $service_name not found"
        return 1
    fi
    
    # HTTP 헬스체크
    for i in {1..5}; do
        if kubectl run health-check-$$ --image=curlimages/curl --rm -it --restart=Never -- \
           curl -f -s http://$service_ip/health >/dev/null 2>&1; then
            echo "✅ Health check passed (attempt $i/5)"
            return 0
        fi
        echo "⚠️ Health check failed (attempt $i/5), retrying..."
        sleep 10
    done
    
    echo "❌ Health check failed after 5 attempts"
    return 1
}

# 트래픽 전환
switch_traffic() {
    local target_version=$1
    
    echo "Switching traffic to $target_version environment..."
    
    # 서비스 셀렉터 업데이트
    kubectl patch service ${APP_NAME}-service -n $NAMESPACE -p \
        "{\"spec\":{\"selector\":{\"version\":\"$target_version\"}}}"
    
    if [ $? -eq 0 ]; then
        echo "✅ Traffic switched to $target_version"
        
        # 전환 확인
        sleep 5
        local current_version=$(get_active_version)
        if [ "$current_version" = "$target_version" ]; then
            echo "✅ Traffic switch confirmed"
            return 0
        else
            echo "❌ Traffic switch verification failed"
            return 1
        fi
    else
        echo "❌ Failed to switch traffic"
        return 1
    fi
}

# 이전 버전 정리
cleanup_old_version() {
    local old_version=$1
    
    echo "Scaling down old version ($old_version)..."
    kubectl scale deployment app-${old_version} -n $NAMESPACE --replicas=0
    
    echo "✅ Old version scaled down"
}

# 롤백 실행
rollback() {
    local current_active=$(get_active_version)
    local rollback_target=$(get_inactive_version)
    
    echo "Rolling back from $current_active to $rollback_target..."
    
    # 이전 버전 다시 스케일업
    kubectl scale deployment app-${rollback_target} -n $NAMESPACE --replicas=3
    kubectl rollout status deployment/app-${rollback_target} -n $NAMESPACE --timeout=${TIMEOUT}s
    
    # 트래픽 다시 전환
    switch_traffic $rollback_target
    
    # 실패한 버전 정리
    cleanup_old_version $current_active
    
    echo "✅ Rollback completed"
}

# 메인 배포 프로세스
main_deploy() {
    echo "=== Blue/Green Deployment Started ==="
    echo "Target version: $NEW_VERSION"
    
    local active_version=$(get_active_version)
    local inactive_version=$(get_inactive_version)
    
    echo "Current active: $active_version"
    echo "Deploying to: $inactive_version"
    
    # 1. 새 버전 배포
    if ! deploy_new_version; then
        echo "❌ Deployment failed, aborting"
        exit 1
    fi
    
    # 2. 헬스체크
    if ! health_check $inactive_version; then
        echo "❌ Health check failed, aborting"
        echo "Manual cleanup required for $inactive_version environment"
        exit 1
    fi
    
    # 3. 트래픽 전환 확인
    echo "Ready to switch traffic from $active_version to $inactive_version"
    read -p "Proceed with traffic switch? (y/N): " -n 1 -r
    echo
    
    if [[ $REPLY =~ ^[Yy]$ ]]; then
        # 4. 트래픽 전환
        if switch_traffic $inactive_version; then
            # 5. 성공 후 정리
            echo "Waiting 60 seconds before cleanup..."
            sleep 60
            cleanup_old_version $active_version
            echo "✅ Blue/Green deployment completed successfully"
        else
            echo "❌ Traffic switch failed, manual intervention required"
            exit 1
        fi
    else
        echo "Deployment aborted by user"
        echo "New version is running in $inactive_version environment for testing"
    fi
}

# 명령어 처리
case "${2:-deploy}" in
    deploy)
        main_deploy
        ;;
    rollback)
        rollback
        ;;
    status)
        echo "Active version: $(get_active_version)"
        echo "Available versions: blue, green"
        kubectl get deployments -n $NAMESPACE -l app=$APP_NAME
        ;;
    switch)
        target_version=$3
        if [ -z "$target_version" ]; then
            echo "Usage: $0 <version> switch <blue|green>"
            exit 1
        fi
        switch_traffic $target_version
        ;;
    *)
        echo "Usage: $0 <new-version> {deploy|rollback|status|switch <blue|green>}"
        exit 1
        ;;
esac
```

## 🐤 카나리 배포

### Istio 기반 카나리 배포

```yaml
# canary-deployment-istio.yaml
---
# 메인 애플리케이션 Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
      version: v1
  template:
    metadata:
      labels:
        app: web-app
        version: v1
    spec:
      containers:
      - name: web-app
        image: registry.company.com/web-app:v1.0.0
        ports:
        - containerPort: 8080

---
# 카나리 버전 Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v2
  namespace: production
spec:
  replicas: 1  # 작은 규모로 시작
  selector:
    matchLabels:
      app: web-app
      version: v2
  template:
    metadata:
      labels:
        app: web-app
        version: v2
    spec:
      containers:
      - name: web-app
        image: registry.company.com/web-app:v2.0.0
        ports:
        - containerPort: 8080

---
# Service (모든 버전 포함)
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  namespace: production
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080

---
# Istio VirtualService (트래픽 분할)
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: web-app-vs
  namespace: production
spec:
  hosts:
  - web-app-service
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: web-app-service
        subset: v2
  - route:
    - destination:
        host: web-app-service
        subset: v1
      weight: 90
    - destination:
        host: web-app-service
        subset: v2
      weight: 10  # 10% 트래픽을 카나리로

---
# Istio DestinationRule
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: web-app-dr
  namespace: production
spec:
  host: web-app-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

### NGINX 기반 카나리 배포

```yaml
# canary-deployment-nginx.yaml
---
# 메인 Ingress (90% 트래픽)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-main
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/canary: "false"
spec:
  ingressClassName: nginx
  rules:
  - host: app.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v1-service
            port:
              number: 80

---
# 카나리 Ingress (10% 트래픽)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-canary
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
    nginx.ingress.kubernetes.io/canary-by-header-value: "always"
    nginx.ingress.kubernetes.io/canary-by-cookie: "canary"
spec:
  ingressClassName: nginx
  rules:
  - host: app.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v2-service
            port:
              number: 80

---
# v1 Service
apiVersion: v1
kind: Service
metadata:
  name: app-v1-service
  namespace: production
spec:
  selector:
    app: web-app
    version: v1
  ports:
  - port: 80
    targetPort: 8080

---
# v2 Service
apiVersion: v1
kind: Service
metadata:
  name: app-v2-service
  namespace: production
spec:
  selector:
    app: web-app
    version: v2
  ports:
  - port: 80
    targetPort: 8080
```

### 카나리 배포 자동화

```bash
#!/bin/bash
# canary-deployment.sh

NAMESPACE="production"
APP_NAME="web-app"
CANARY_VERSION=$1
CANARY_PERCENTAGE=${2:-10}

# 카나리 배포 시작
start_canary() {
    echo "Starting canary deployment..."
    echo "Canary version: $CANARY_VERSION"
    echo "Traffic percentage: $CANARY_PERCENTAGE%"
    
    # 카나리 deployment 생성/업데이트
    cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}-canary
  namespace: $NAMESPACE
spec:
  replicas: 1
  selector:
    matchLabels:
      app: $APP_NAME
      version: canary
  template:
    metadata:
      labels:
        app: $APP_NAME
        version: canary
    spec:
      containers:
      - name: $APP_NAME
        image: registry.company.com/${APP_NAME}:${CANARY_VERSION}
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
EOF
    
    # 배포 완료 대기
    kubectl rollout status deployment/${APP_NAME}-canary -n $NAMESPACE --timeout=300s
    
    if [ $? -eq 0 ]; then
        echo "✅ Canary deployment ready"
        return 0
    else
        echo "❌ Canary deployment failed"
        return 1
    fi
}

# 트래픽 비율 조정
adjust_traffic() {
    local percentage=$1
    
    echo "Adjusting canary traffic to $percentage%..."
    
    # NGINX Ingress의 canary weight 업데이트
    kubectl patch ingress ${APP_NAME}-canary -n $NAMESPACE -p \
        "{\"metadata\":{\"annotations\":{\"nginx.ingress.kubernetes.io/canary-weight\":\"$percentage\"}}}"
    
    echo "✅ Traffic adjusted to $percentage%"
}

# 메트릭 모니터링
monitor_canary() {
    echo "Monitoring canary deployment..."
    
    # 에러율 확인 (Prometheus 쿼리 예시)
    local error_rate=$(curl -s "http://prometheus:9090/api/v1/query?query=rate(http_requests_total{job=\"$APP_NAME\",status=~\"5..\"}[5m])/rate(http_requests_total{job=\"$APP_NAME\"}[5m])*100" | jq -r '.data.result[0].value[1]' 2>/dev/null || echo "0")
    
    # 응답 시간 확인
    local response_time=$(curl -s "http://prometheus:9090/api/v1/query?query=histogram_quantile(0.95,rate(http_request_duration_seconds_bucket{job=\"$APP_NAME\"}[5m]))" | jq -r '.data.result[0].value[1]' 2>/dev/null || echo "0")
    
    echo "Error rate: ${error_rate}%"
    echo "95th percentile response time: ${response_time}s"
    
    # 임계값 확인
    if (( $(echo "$error_rate > 5" | bc -l) )); then
        echo "⚠️ High error rate detected: $error_rate%"
        return 1
    fi
    
    if (( $(echo "$response_time > 1" | bc -l) )); then
        echo "⚠️ High response time detected: ${response_time}s"
        return 1
    fi
    
    echo "✅ Metrics within acceptable range"
    return 0
}

# 점진적 트래픽 증가
gradual_rollout() {
    local stages=(10 25 50 75 100)
    
    for stage in "${stages[@]}"; do
        echo "=== Increasing traffic to $stage% ==="
        
        adjust_traffic $stage
        
        # 모니터링 대기 시간
        echo "Waiting 5 minutes for metrics collection..."
        sleep 300
        
        # 메트릭 확인
        if ! monitor_canary; then
            echo "❌ Metrics check failed at $stage%. Rolling back..."
            rollback_canary
            return 1
        fi
        
        if [ "$stage" -eq 100 ]; then
            echo "✅ Canary rollout completed successfully"
            promote_canary
        else
            echo "✅ Stage $stage% completed successfully"
        fi
    done
}

# 카나리 승격 (100% 트래픽으로 전환 후 기존 버전 제거)
promote_canary() {
    echo "Promoting canary to production..."
    
    # 기존 메인 deployment 이미지 업데이트
    local canary_image=$(kubectl get deployment ${APP_NAME}-canary -n $NAMESPACE -o jsonpath='{.spec.template.spec.containers[0].image}')
    
    kubectl set image deployment/${APP_NAME} -n $NAMESPACE ${APP_NAME}=$canary_image
    kubectl rollout status deployment/${APP_NAME} -n $NAMESPACE --timeout=300s
    
    # 카나리 ingress 제거
    kubectl delete ingress ${APP_NAME}-canary -n $NAMESPACE
    
    # 카나리 deployment 제거
    kubectl delete deployment ${APP_NAME}-canary -n $NAMESPACE
    
    echo "✅ Canary promoted to production"
}

# 카나리 롤백
rollback_canary() {
    echo "Rolling back canary deployment..."
    
    # 카나리 트래픽을 0으로 설정
    adjust_traffic 0
    
    # 잠시 대기 후 정리
    sleep 30
    
    # 카나리 리소스 정리
    kubectl delete ingress ${APP_NAME}-canary -n $NAMESPACE 2>/dev/null || true
    kubectl delete deployment ${APP_NAME}-canary -n $NAMESPACE 2>/dev/null || true
    
    echo "✅ Canary rollback completed"
}

# 메인 실행
case "${3:-start}" in
    start)
        start_canary
        adjust_traffic $CANARY_PERCENTAGE
        ;;
    monitor)
        monitor_canary
        ;;
    adjust)
        adjust_traffic $CANARY_PERCENTAGE
        ;;
    gradual)
        gradual_rollout
        ;;
    promote)
        promote_canary
        ;;
    rollback)
        rollback_canary
        ;;
    *)
        echo "Usage: $0 <canary-version> <percentage> {start|monitor|adjust|gradual|promote|rollback}"
        echo "Example: $0 v2.0.0 10 start"
        exit 1
        ;;
esac
```

## 📋 배포 전략 비교

### 전략별 특징 비교

| 전략 | 다운타임 | 리소스 사용량 | 롤백 속도 | 위험도 | 복잡도 |
|------|----------|---------------|-----------|--------|--------|
| **롤링 업데이트** | 없음 | 낮음 | 빠름 | 중간 | 낮음 |
| **블루/그린** | 없음 | 높음 (2배) | 매우 빠름 | 낮음 | 중간 |
| **카나리** | 없음 | 중간 | 빠름 | 매우 낮음 | 높음 |
| **재생성** | 있음 | 낮음 | 빠름 | 높음 | 낮음 |

### 사용 권장 사항

**롤링 업데이트**: 
- 일반적인 애플리케이션 업데이트
- 리소스가 제한적인 환경
- 빠른 반복 개발

**블루/그린**: 
- 중요한 서비스 업데이트
- 즉시 롤백이 필요한 경우
- 충분한 리소스가 있는 환경

**카나리**: 
- 고위험 업데이트
- 새로운 기능 테스트
- 단계적 검증이 필요한 경우

---

> 💡 **실전 경험**: 배포 전략 선택은 애플리케이션의 특성과 비즈니스 요구사항에 따라 결정해야 합니다. 온프렘 환경에서는 리소스 제약을 고려하여 롤링 업데이트를 기본으로 하되, 중요한 서비스에는 블루/그린이나 카나리 배포를 적용하는 것을 권장합니다. 자동화 스크립트와 모니터링을 통해 배포 위험을 최소화하세요.

태그: #deployment #rolling-update #blue-green #canary #strategy #automation #onprem
