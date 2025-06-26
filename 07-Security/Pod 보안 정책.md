# Pod 보안 정책

## 🎯 개요

쿠버네티스 Pod의 보안 강화를 위한 정책 설정, SecurityContext 구성, Pod Security Standards 적용, 그리고 온프렘 환경에서의 보안 모범 사례를 다룹니다.

## 🛡️ Pod Security 개념

### Pod Security 레벨

```
Privileged (특권) ← 가장 낮은 보안
    ↓
Baseline (기본) ← 일반적인 제한
    ↓  
Restricted (제한) ← 가장 높은 보안
```

### 보안 계층 구조

```
┌─────────────────────────┐
│   Pod Security Policy   │ ← 클러스터 레벨 정책 (deprecated)
├─────────────────────────┤
│  Pod Security Standards │ ← 새로운 표준 (v1.23+)
├─────────────────────────┤
│    Security Context     │ ← Pod/Container 레벨 설정
├─────────────────────────┤
│      AppArmor/SELinux   │ ← OS 레벨 보안
└─────────────────────────┘
```

## 🔐 SecurityContext 설정

### Pod 레벨 SecurityContext

```yaml
# pod-security-contexts.yaml
---
# 기본 보안 설정
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod-basic
spec:
  securityContext:
    # 루트 사용자 금지
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 2000
    fsGroup: 3000
    
    # 권한 상승 방지
    allowPrivilegeEscalation: false
    
    # 읽기 전용 루트 파일시스템
    readOnlyRootFilesystem: true
    
    # 보안 프로파일
    seccompProfile:
      type: RuntimeDefault
    
  containers:
  - name: app
    image: nginx:1.20
    volumeMounts:
    - name: tmp-volume
      mountPath: /tmp
    - name: cache-volume  
      mountPath: /var/cache/nginx
    - name: run-volume
      mountPath: /var/run
  
  volumes:
  - name: tmp-volume
    emptyDir: {}
  - name: cache-volume
    emptyDir: {}
  - name: run-volume
    emptyDir: {}

---
# 고급 보안 설정 (데이터베이스 예제)
apiVersion: v1
kind: Pod
metadata:
  name: secure-database
  annotations:
    container.apparmor.security.beta.kubernetes.io/mysql: runtime/default
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 999
    runAsGroup: 999
    fsGroup: 999
    fsGroupChangePolicy: "OnRootMismatch"
    
    # 추가 그룹 설정
    supplementalGroups: [1000, 2000]
    
    # SELinux 설정
    seLinuxOptions:
      level: "s0:c123,c456"
      
  containers:
  - name: mysql
    image: mysql:8.0
    securityContext:
      # 컨테이너별 추가 제한
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
        add:
        - CHOWN
        - DAC_OVERRIDE
        - SETUID
        - SETGID
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-secret
          key: password
    volumeMounts:
    - name: mysql-data
      mountPath: /var/lib/mysql
    - name: mysql-tmp
      mountPath: /tmp
    - name: mysql-run
      mountPath: /var/run/mysqld
      
  volumes:
  - name: mysql-data
    persistentVolumeClaim:
      claimName: mysql-pvc
  - name: mysql-tmp
    emptyDir: {}
  - name: mysql-run
    emptyDir: {}

---
# 웹 애플리케이션 보안 설정
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-web-app
  template:
    metadata:
      labels:
        app: secure-web-app
      annotations:
        container.apparmor.security.beta.kubernetes.io/web-app: runtime/default
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
        
      containers:
      - name: web-app
        image: nginx:1.20-alpine
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
          seccompProfile:
            type: RuntimeDefault
            
        ports:
        - containerPort: 8080
          protocol: TCP
          
        volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx/conf.d
          readOnly: true
        - name: tmp-volume
          mountPath: /tmp
        - name: var-cache
          mountPath: /var/cache/nginx
        - name: var-run
          mountPath: /var/run
          
        resources:
          limits:
            memory: "256Mi"
            cpu: "200m"
          requests:
            memory: "128Mi"
            cpu: "100m"
            
      volumes:
      - name: nginx-conf
        configMap:
          name: nginx-config
      - name: tmp-volume
        emptyDir: {}
      - name: var-cache
        emptyDir: {}
      - name: var-run
        emptyDir: {}
```

## 🔒 Pod Security Standards

### Namespace별 Pod Security 적용

```yaml
# pod-security-standards.yaml
---
# Restricted 네임스페이스 (최고 보안 레벨)
apiVersion: v1
kind: Namespace
metadata:
  name: production-secure
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
# Baseline 네임스페이스 (일반 보안 레벨)  
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
# Privileged 네임스페이스 (시스템 컴포넌트용)
apiVersion: v1
kind: Namespace
metadata:
  name: system-privileged
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

### Pod Security Standards 검증

```bash
#!/bin/bash
# pod-security-validator.sh

validate_pod_security() {
    local namespace=$1
    local pod_manifest=$2
    
    echo "=== Validating Pod Security for namespace: $namespace ==="
    
    # 네임스페이스의 Pod Security 레벨 확인
    enforce_level=$(kubectl get namespace $namespace -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}')
    audit_level=$(kubectl get namespace $namespace -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/audit}')
    warn_level=$(kubectl get namespace $namespace -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/warn}')
    
    echo "Namespace $namespace security levels:"
    echo "  Enforce: ${enforce_level:-"not set"}"
    echo "  Audit: ${audit_level:-"not set"}"
    echo "  Warn: ${warn_level:-"not set"}"
    echo
    
    # Pod 매니페스트 검증 (dry-run)
    if [ -n "$pod_manifest" ]; then
        echo "Validating Pod manifest: $pod_manifest"
        kubectl apply -f "$pod_manifest" -n "$namespace" --dry-run=server --warnings-as-errors 2>&1 | \
        grep -E "(Warning|Error|Forbidden)" || echo "✓ Pod security validation passed"
    fi
}

# 클러스터의 모든 네임스페이스 Pod Security 상태 확인
check_cluster_security() {
    echo "=== Cluster Pod Security Overview ==="
    echo
    
    printf "%-20s %-12s %-12s %-12s\n" "NAMESPACE" "ENFORCE" "AUDIT" "WARN"
    printf "%-20s %-12s %-12s %-12s\n" "----------" "-------" "-----" "----"
    
    kubectl get namespaces -o custom-columns="NAME:.metadata.name,ENFORCE:.metadata.labels.pod-security\.kubernetes\.io/enforce,AUDIT:.metadata.labels.pod-security\.kubernetes\.io/audit,WARN:.metadata.labels.pod-security\.kubernetes\.io/warn" --no-headers | \
    while read ns enforce audit warn; do
        printf "%-20s %-12s %-12s %-12s\n" "$ns" "${enforce:-"none"}" "${audit:-"none"}" "${warn:-"none"}"
    done
}

# 보안 위반 Pod 검색
find_security_violations() {
    echo "=== Finding Security Violations ==="
    echo
    
    # 권한 상승이 허용된 Pod 찾기
    echo "Pods with privileged containers:"
    kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.containers[*].securityContext.privileged}{"\n"}{end}' | \
    grep true | awk '{print $1 "/" $2}'
    echo
    
    # 루트로 실행되는 Pod 찾기
    echo "Pods running as root:"
    kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.securityContext.runAsUser}{"\t"}{.spec.containers[*].securityContext.runAsUser}{"\n"}{end}' | \
    awk '$3 == 0 || $4 == 0 {print $1 "/" $2}'
    echo
    
    # 호스트 네트워크를 사용하는 Pod 찾기
    echo "Pods using host network:"
    kubectl get pods --all-namespaces -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.hostNetwork}{"\n"}{end}' | \
    grep true | awk '{print $1 "/" $2}'
}

case "${1:-check}" in
    validate)
        validate_pod_security "$2" "$3"
        ;;
    check)
        check_cluster_security
        ;;
    violations)
        find_security_violations
        ;;
    *)
        echo "Usage: $0 {validate <namespace> [pod-manifest]|check|violations}"
        exit 1
        ;;
esac
```

## 🛠️ 리소스 제한과 보안

### ResourceQuota와 LimitRange

```yaml
# resource-security-limits.yaml
---
# 네임스페이스별 리소스 할당량
apiVersion: v1
kind: ResourceQuota
metadata:
  name: security-quota
  namespace: production
spec:
  hard:
    # 기본 리소스 제한
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20" 
    limits.memory: 40Gi
    
    # Pod 수 제한
    pods: "50"
    
    # 스토리지 제한
    requests.storage: "100Gi"
    persistentvolumeclaims: "10"
    
    # 보안 관련 제한
    count/secrets: "10"
    count/configmaps: "20"
    count/services: "10"

---
# Pod별 리소스 제한
apiVersion: v1
kind: LimitRange
metadata:
  name: security-limits
  namespace: production
spec:
  limits:
  # Pod 레벨 제한
  - type: Pod
    max:
      cpu: "4"
      memory: "8Gi"
    min:
      cpu: "10m"
      memory: "16Mi"
      
  # 컨테이너 레벨 제한
  - type: Container
    default:
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "2"
      memory: "4Gi"
    min:
      cpu: "10m"
      memory: "16Mi"
      
  # PVC 제한
  - type: PersistentVolumeClaim
    max:
      storage: "50Gi"
    min:
      storage: "1Gi"

---
# 높은 보안 수준의 LimitRange
apiVersion: v1
kind: LimitRange
metadata:
  name: restricted-limits
  namespace: production-secure
spec:
  limits:
  - type: Container
    default:
      cpu: "100m"
      memory: "128Mi"
    defaultRequest:
      cpu: "50m"
      memory: "64Mi"
    max:
      cpu: "500m"
      memory: "512Mi"
    min:
      cpu: "10m"
      memory: "32Mi"
    # 임시 스토리지 제한
    maxLimitRequestRatio:
      cpu: "10"
      memory: "4"
```

### 네트워크 보안 정책

```yaml
# network-security-policies.yaml
---
# 기본 Deny All 정책
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# 웹 애플리케이션 네트워크 정책
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-app-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  # Ingress Controller로부터만 트래픽 허용
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
      
  egress:
  # 데이터베이스 접근 허용
  - to:
    - podSelector:
        matchLabels:
          app: mysql
    ports:
    - protocol: TCP
      port: 3306
      
  # DNS 해결 허용
  - to: []
    ports:
    - protocol: UDP
      port: 53

---
# 데이터베이스 네트워크 정책
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: mysql
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  # 웹 애플리케이션으로부터만 접근 허용
  - from:
    - podSelector:
        matchLabels:
          app: web-app
    ports:
    - protocol: TCP
      port: 3306
      
  egress:
  # DNS 해결 허용
  - to: []
    ports:
    - protocol: UDP
      port: 53
```

## 🔍 보안 모니터링과 감사

### 보안 이벤트 모니터링

```bash
#!/bin/bash
# security-monitoring.sh

# 보안 관련 이벤트 모니터링
monitor_security_events() {
    echo "=== Security Events Monitoring ==="
    echo "Timestamp: $(date)"
    echo
    
    # 최근 보안 관련 이벤트
    echo "=== Recent Security Events ==="
    kubectl get events --all-namespaces --sort-by='.lastTimestamp' | \
    grep -E "(Warning|Failed|Error|Forbidden|SecurityContext|Privileged|Root)" | tail -20
    echo
    
    # Pod Security 위반 이벤트
    echo "=== Pod Security Violations ==="
    kubectl get events --all-namespaces --field-selector reason=FailedCreate | \
    grep -i "security\|privilege\|forbidden" | tail -10
    echo
    
    # 실패한 인증/인가 이벤트
    echo "=== Authentication/Authorization Failures ==="
    kubectl get events --all-namespaces | grep -E "(Forbidden|Unauthorized|RBAC)" | tail -10
}

# 권한이 높은 Pod 검색
find_privileged_pods() {
    echo "=== Privileged Pods Analysis ==="
    echo
    
    echo "Pods with privileged containers:"
    kubectl get pods --all-namespaces -o json | jq -r '
    .items[] | 
    select(.spec.containers[]?.securityContext?.privileged == true) |
    "\(.metadata.namespace)/\(.metadata.name)"'
    echo
    
    echo "Pods running as root (UID 0):"
    kubectl get pods --all-namespaces -o json | jq -r '
    .items[] | 
    select(
        (.spec.securityContext?.runAsUser == 0) or 
        (.spec.containers[]?.securityContext?.runAsUser == 0)
    ) |
    "\(.metadata.namespace)/\(.metadata.name)"'
    echo
    
    echo "Pods with dangerous capabilities:"
    kubectl get pods --all-namespaces -o json | jq -r '
    .items[] | 
    select(
        .spec.containers[]?.securityContext?.capabilities?.add[]? | 
        test("SYS_ADMIN|NET_ADMIN|SYS_PTRACE|SYS_MODULE")
    ) |
    "\(.metadata.namespace)/\(.metadata.name)"'
}

# 보안 설정 체크
security_configuration_check() {
    echo "=== Security Configuration Check ==="
    echo
    
    # Pod Security Standards 설정 확인
    echo "Namespaces without Pod Security Standards:"
    kubectl get namespaces -o json | jq -r '
    .items[] | 
    select(
        (.metadata.labels["pod-security.kubernetes.io/enforce"] // "none") == "none"
    ) |
    .metadata.name'
    echo
    
    # RBAC 설정 확인
    echo "ClusterRoleBindings with cluster-admin:"
    kubectl get clusterrolebindings -o json | jq -r '
    .items[] |
    select(.roleRef.name == "cluster-admin") |
    "\(.metadata.name): \(.subjects[]?.name // .subjects[]?.metadata.name)"'
    echo
    
    # 기본 ServiceAccount 사용 Pod
    echo "Pods using default ServiceAccount:"
    kubectl get pods --all-namespaces -o json | jq -r '
    .items[] |
    select(.spec.serviceAccountName == "default" or .spec.serviceAccountName == null) |
    "\(.metadata.namespace)/\(.metadata.name)"' | head -10
}

case "${1:-monitor}" in
    monitor)
        monitor_security_events
        ;;
    privileged)
        find_privileged_pods
        ;;
    config)
        security_configuration_check
        ;;
    *)
        echo "Usage: $0 {monitor|privileged|config}"
        exit 1
        ;;
esac
```

### 보안 정책 템플릿

```yaml
# security-policy-templates.yaml
---
# 일반 애플리케이션용 보안 템플릿
apiVersion: v1
kind: Pod
metadata:
  name: secure-app-template
  annotations:
    # AppArmor 프로파일 적용
    container.apparmor.security.beta.kubernetes.io/app: runtime/default
spec:
  # ServiceAccount 지정 (default 사용 금지)
  serviceAccountName: app-service-account
  automountServiceAccountToken: false
  
  securityContext:
    # 루트 사용자 금지
    runAsNonRoot: true
    runAsUser: 1001
    runAsGroup: 1001
    fsGroup: 1001
    
    # 권한 상승 방지
    allowPrivilegeEscalation: false
    
    # 읽기 전용 루트 파일시스템
    readOnlyRootFilesystem: true
    
    # Seccomp 프로파일
    seccompProfile:
      type: RuntimeDefault
      
  containers:
  - name: app
    image: app:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
        
    # 리소스 제한
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
      limits:
        memory: "128Mi"
        cpu: "100m"
        
    # 헬스체크
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      
    # 임시 디렉토리 마운트
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/cache
      
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}

---
# 데이터베이스용 보안 템플릿
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: secure-database-template
spec:
  serviceName: database
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
      annotations:
        container.apparmor.security.beta.kubernetes.io/database: runtime/default
    spec:
      serviceAccountName: database-service-account
      
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        runAsGroup: 999
        fsGroup: 999
        fsGroupChangePolicy: "OnRootMismatch"
        
      containers:
      - name: database
        image: postgres:13-alpine
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
            add:
            - CHOWN
            - DAC_OVERRIDE
            - SETUID
            - SETGID
            
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: password
              
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "200m"
            
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        - name: tmp
          mountPath: /tmp
        - name: run
          mountPath: /var/run/postgresql
          
      volumes:
      - name: tmp
        emptyDir: {}
      - name: run
        emptyDir: {}
        
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: secure-storage
      resources:
        requests:
          storage: 10Gi
```

## 📋 보안 체크리스트

### 배포 전 체크리스트
- [ ] **SecurityContext 설정**
  - [ ] runAsNonRoot: true 설정
  - [ ] 적절한 runAsUser/runAsGroup 설정
  - [ ] allowPrivilegeEscalation: false 설정
  - [ ] readOnlyRootFilesystem: true 설정 (가능한 경우)
  
- [ ] **Capabilities 제한**
  - [ ] 모든 capabilities DROP
  - [ ] 필요한 capabilities만 ADD
  
- [ ] **리소스 제한**
  - [ ] CPU/Memory requests와 limits 설정
  - [ ] 적절한 리소스 할당량
  
- [ ] **네트워크 보안**
  - [ ] NetworkPolicy 적용
  - [ ] 필요한 트래픽만 허용
  
- [ ] **ServiceAccount**
  - [ ] default ServiceAccount 사용 금지
  - [ ] 최소 권한 원칙 적용

### 운영 중 체크리스트
- [ ] **정기 보안 스캔**
  - [ ] 컨테이너 이미지 취약점 스캔
  - [ ] 권한 설정 검토
  
- [ ] **모니터링**
  - [ ] 보안 이벤트 모니터링
  - [ ] 비정상적인 권한 사용 감지
  
- [ ] **업데이트 관리**
  - [ ] 보안 패치 적용
  - [ ] 이미지 정기 업데이트

---

> 💡 **실전 경험**: Pod 보안 설정은 애플리케이션 기능과 보안 사이의 균형입니다. 처음에는 제한적으로 시작해서 필요에 따라 점진적으로 권한을 추가하는 방식을 권장합니다. 특히 readOnlyRootFilesystem 설정은 많은 애플리케이션에서 추가 설정이 필요할 수 있습니다.

태그: #security #pod-security #securitycontext #rbac #onprem
