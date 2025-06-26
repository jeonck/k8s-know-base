# RBAC 설계와 구현

## 🎯 개요

Role-Based Access Control(RBAC)은 쿠버네티스 보안의 핵심입니다. 이 가이드는 온프렘 환경에서 안전하고 확장 가능한 RBAC 체계를 설계하고 구현하는 방법을 다룹니다.

## 🏗️ RBAC 아키텍처

### RBAC 구성요소

```
    ┌─────────────┐
    │   Subject   │ (User, Group, ServiceAccount)
    └─────┬───────┘
          │ bound to
    ┌─────▼───────┐
    │ RoleBinding │ or ClusterRoleBinding
    └─────┬───────┘
          │ grants
    ┌─────▼───────┐
    │    Role     │ or ClusterRole
    └─────┬───────┘
          │ contains
    ┌─────▼───────┐
    │    Rules    │ (API Groups, Resources, Verbs)
    └─────────────┘
```

### 권한 계층 구조

```yaml
# 권한 레벨별 분류
레벨 1 - Viewer:
  - 읽기 전용 권한
  - 로그 조회
  - 메트릭 확인

레벨 2 - Developer:
  - 특정 네임스페이스 내 Pod, Service 관리
  - ConfigMap, Secret 읽기
  - 로그 및 디버깅

레벨 3 - DevOps:
  - 배포 관리
  - 네임스페이스 관리
  - 리소스 할당량 관리

레벨 4 - Admin:
  - 클러스터 전체 관리
  - 노드 관리
  - RBAC 정책 관리

레벨 5 - Cluster Admin:
  - 모든 권한
  - 보안 정책 관리
  - 시스템 구성 변경
```

## 🔐 기본 RBAC 설정

### 네임스페이스별 환경 분리

```bash
# 환경별 네임스페이스 생성
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production
kubectl create namespace monitoring
kubectl create namespace system
```

### 기본 ClusterRole 정의

```yaml
# cluster-roles.yaml
---
# 읽기 전용 역할
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: viewer
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "endpoints", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "daemonsets", "statefulsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses", "networkpolicies"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list"]

---
# 개발자 역할
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "endpoints", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "daemonsets", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods/exec", "pods/portforward"]
  verbs: ["create"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

---
# DevOps 역할
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: devops
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["roles", "rolebindings"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses", "networkpolicies"]
  verbs: ["*"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses", "volumeattachments"]
  verbs: ["get", "list", "watch"]

---
# 모니터링 역할
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/proxy", "services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["extensions"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
```

## 👥 사용자 및 그룹 관리

### 인증서 기반 사용자 생성

```bash
#!/bin/bash
# create-user.sh

USER_NAME=$1
NAMESPACE=$2
ROLE=$3

if [ -z "$USER_NAME" ] || [ -z "$NAMESPACE" ] || [ -z "$ROLE" ]; then
    echo "Usage: $0 <username> <namespace> <role>"
    echo "Roles: viewer, developer, devops"
    exit 1
fi

# 개인키 생성
openssl genrsa -out ${USER_NAME}.key 2048

# 인증서 서명 요청 생성
openssl req -new -key ${USER_NAME}.key -out ${USER_NAME}.csr -subj "/CN=${USER_NAME}/O=company"

# 인증서 서명 요청을 쿠버네티스에 제출
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: ${USER_NAME}
spec:
  request: $(cat ${USER_NAME}.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF

# 인증서 승인
kubectl certificate approve ${USER_NAME}

# 인증서 추출
kubectl get csr ${USER_NAME} -o jsonpath='{.status.certificate}' | base64 -d > ${USER_NAME}.crt

# kubeconfig 생성
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://192.168.1.10:6443 \
  --kubeconfig=${USER_NAME}-kubeconfig

kubectl config set-credentials ${USER_NAME} \
  --client-certificate=${USER_NAME}.crt \
  --client-key=${USER_NAME}.key \
  --embed-certs=true \
  --kubeconfig=${USER_NAME}-kubeconfig

kubectl config set-context ${USER_NAME}@kubernetes \
  --cluster=kubernetes \
  --user=${USER_NAME} \
  --namespace=${NAMESPACE} \
  --kubeconfig=${USER_NAME}-kubeconfig

kubectl config use-context ${USER_NAME}@kubernetes --kubeconfig=${USER_NAME}-kubeconfig

# RoleBinding 생성
kubectl create rolebinding ${USER_NAME}-${ROLE} \
  --clusterrole=${ROLE} \
  --user=${USER_NAME} \
  --namespace=${NAMESPACE}

echo "User ${USER_NAME} created with ${ROLE} role in ${NAMESPACE} namespace"
echo "kubeconfig file: ${USER_NAME}-kubeconfig"
```

### ServiceAccount 기반 애플리케이션 권한

```yaml
# app-service-accounts.yaml
---
# 개발 환경 애플리케이션 SA
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-dev-sa
  namespace: development
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-dev-binding
  namespace: development
subjects:
- kind: ServiceAccount
  name: app-dev-sa
  namespace: development
roleRef:
  kind: ClusterRole
  name: developer
  apiGroup: rbac.authorization.k8s.io

---
# 운영 환경 애플리케이션 SA (제한적 권한)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-prod-sa
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: production-app-role
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-prod-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-prod-sa
  namespace: production
roleRef:
  kind: Role
  name: production-app-role
  apiGroup: rbac.authorization.k8s.io
```

## 🔒 고급 RBAC 패턴

### 네임스페이스별 관리자 설정

```yaml
# namespace-admin.yaml
---
# 네임스페이스별 관리자 역할
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: namespace-admin
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["apps"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["networking.k8s.io"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["roles", "rolebindings"]
  verbs: ["*"]
- apiGroups: ["batch"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-admin-binding
  namespace: development
subjects:
- kind: User
  name: dev-team-lead
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: namespace-admin
  apiGroup: rbac.authorization.k8s.io
```

### 리소스별 세분화된 권한

```yaml
# granular-permissions.yaml
---
# 시크릿 읽기 전용 역할
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
  resourceNames: ["app-config", "db-credentials"]  # 특정 시크릿만

---
# 특정 라벨의 Pod만 관리
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: specific-pod-manager
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods/exec", "pods/portforward"]
  verbs: ["create"]

---
# 바인딩 시 라벨 셀렉터 적용 (추가 설정 필요)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: specific-pod-binding
  namespace: production
subjects:
- kind: User
  name: app-developer
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: specific-pod-manager
  apiGroup: rbac.authorization.k8s.io
```

### 시간 기반 접근 제어 (with Admission Controller)

```yaml
# time-based-access.yaml
# 이는 커스텀 Admission Controller와 함께 사용
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: emergency-access
  annotations:
    access.company.com/valid-from: "2024-01-01T00:00:00Z"
    access.company.com/valid-until: "2024-12-31T23:59:59Z"
    access.company.com/time-restriction: "business-hours"  # 09:00-18:00
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch", "delete"]
```

## 🛡️ 보안 강화 설정

### Pod Security Standards 적용

```yaml
# pod-security-policy.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Network Policy와 RBAC 연동

```yaml
# rbac-network-policy.yaml
---
# 네트워크 정책 관리 역할
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: network-policy-manager
rules:
- apiGroups: ["networking.k8s.io"]
  resources: ["networkpolicies"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list", "watch"]

---
# 보안팀만 네트워크 정책 수정 가능
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: security-team-network-binding
subjects:
- kind: Group
  name: security-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: network-policy-manager
  apiGroup: rbac.authorization.k8s.io
```

## 🔍 RBAC 감사 및 모니터링

### 권한 검증 스크립트

```bash
#!/bin/bash
# rbac-audit.sh

echo "=== RBAC Audit Report ==="
echo "Generated: $(date)"
echo

echo "=== Users and ServiceAccounts ==="
kubectl get rolebindings,clusterrolebindings \
  --all-namespaces -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,ROLE:.roleRef.name,SUBJECT:.subjects[*].name,KIND:.subjects[*].kind'

echo -e "\n=== High Privilege Bindings ==="
kubectl get clusterrolebindings -o json | jq -r '
  .items[] | 
  select(.roleRef.name | test("cluster-admin|admin|edit")) |
  "\(.metadata.name): \(.roleRef.name) -> \(.subjects[]?.name // "N/A")"'

echo -e "\n=== ServiceAccounts with ClusterRole bindings ==="
kubectl get clusterrolebindings -o json | jq -r '
  .items[] | 
  select(.subjects[]?.kind == "ServiceAccount") |
  "\(.metadata.name): \(.roleRef.name) -> \(.subjects[]?.namespace)/\(.subjects[]?.name)"'

echo -e "\n=== Orphaned RoleBindings ==="
kubectl get rolebindings --all-namespaces -o json | jq -r '
  .items[] | 
  select(.subjects[]?.kind == "User") |
  "\(.metadata.namespace)/\(.metadata.name): \(.subjects[]?.name)"'
```

### 권한 사용량 모니터링

```yaml
# rbac-monitoring.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: rbac-metrics
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: rbac-exporter
  endpoints:
  - port: metrics
    interval: 30s
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rbac-exporter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rbac-exporter
  template:
    metadata:
      labels:
        app: rbac-exporter
    spec:
      serviceAccountName: rbac-exporter
      containers:
      - name: rbac-exporter
        image: quay.io/coreos/kube-rbac-proxy:v0.13.1
        ports:
        - containerPort: 8080
          name: metrics
        args:
        - "--secure-listen-address=0.0.0.0:8080"
        - "--upstream=http://127.0.0.1:8081/"
        - "--logtostderr=true"
        - "--v=10"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rbac-exporter
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: rbac-exporter
rules:
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: rbac-exporter
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: rbac-exporter
subjects:
- kind: ServiceAccount
  name: rbac-exporter
  namespace: monitoring
```

## 🚨 실전 운영 가이드

### 권한 에스컬레이션 방지

```yaml
# prevent-privilege-escalation.yaml
---
# 권한 에스컬레이션 방지 정책
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: limited-admin
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["apps", "extensions"]
  resources: ["*"]
  verbs: ["*"]
# RBAC 리소스는 제외
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
# 노드 관리 제외
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

### 임시 권한 부여 프로세스

```bash
#!/bin/bash
# grant-emergency-access.sh

USER=$1
DURATION=${2:-"1h"}
REASON=$3

if [ -z "$USER" ] || [ -z "$REASON" ]; then
    echo "Usage: $0 <username> [duration] <reason>"
    exit 1
fi

# 임시 권한 부여
kubectl create rolebinding emergency-access-${USER} \
  --clusterrole=cluster-admin \
  --user=${USER} \
  --namespace=production

# 자동 만료를 위한 Job 생성
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: revoke-emergency-access-${USER}
spec:
  schedule: "now + ${DURATION}"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: revoke-access
            image: bitnami/kubectl:latest
            command: 
            - kubectl
            - delete
            - rolebinding
            - emergency-access-${USER}
            - --namespace=production
          restartPolicy: OnFailure
EOF

echo "Emergency access granted to ${USER} for ${DURATION}"
echo "Reason: ${REASON}"
echo "Access will be automatically revoked after ${DURATION}"
```

### 정기 권한 검토

```bash
#!/bin/bash
# quarterly-rbac-review.sh

echo "=== Quarterly RBAC Review ==="
echo "Review Date: $(date)"
echo

# 모든 사용자별 권한 매트릭스 생성
echo "=== User Access Matrix ==="
for binding in $(kubectl get rolebindings,clusterrolebindings --all-namespaces -o name); do
    kubectl describe $binding 2>/dev/null | grep -E "Name:|Namespace:|Role|Subjects:"
    echo "---"
done

# 미사용 ServiceAccount 찾기
echo -e "\n=== Unused ServiceAccounts ==="
kubectl get serviceaccounts --all-namespaces -o json | jq -r '
  .items[] | 
  select(.metadata.name != "default") |
  "\(.metadata.namespace)/\(.metadata.name)"' | while read sa; do
    # 30일 이상 사용되지 않은 SA 찾기 (실제로는 더 복잡한 로직 필요)
    echo "Check: $sa"
done

# 높은 권한 바인딩 리뷰
echo -e "\n=== High Privilege Bindings Review ==="
kubectl get clusterrolebindings -o json | jq -r '
  .items[] | 
  select(.roleRef.name | test("cluster-admin|admin")) |
  "REVIEW REQUIRED: \(.metadata.name) grants \(.roleRef.name) to \(.subjects[]?.name // "N/A")"'
```

## 📋 RBAC 체크리스트

### 초기 설정
- [ ] 기본 ClusterRole 정의
- [ ] 네임스페이스별 분리
- [ ] 최소 권한 원칙 적용
- [ ] ServiceAccount 설정

### 사용자 관리
- [ ] 인증서 기반 사용자 생성
- [ ] 그룹 기반 권한 관리
- [ ] 임시 권한 부여 프로세스
- [ ] 권한 만료 정책

### 모니터링
- [ ] RBAC 감사 로그 활성화
- [ ] 권한 사용량 모니터링
- [ ] 정기 권한 검토
- [ ] 보안 이벤트 알림

### 보안 강화
- [ ] Pod Security Standards 적용
- [ ] Network Policy 연동
- [ ] 권한 에스컬레이션 방지
- [ ] 민감한 리소스 보호

---

> 💡 **실전 경험**: RBAC는 처음에는 복잡해 보이지만, 일관된 네이밍 규칙과 체계적인 역할 분류를 통해 관리 복잡도를 크게 줄일 수 있습니다. 정기적인 권한 검토가 보안 유지의 핵심입니다.

태그: #rbac #security #authentication #authorization #onprem