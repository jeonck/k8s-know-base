# PV와 PVC 관리

## 🎯 개요

쿠버네티스 스토리지의 핵심인 PersistentVolume(PV)과 PersistentVolumeClaim(PVC)의 생명주기 관리, 최적화 전략, 그리고 온프렘 환경에서의 실전 운영 방법을 다룹니다.

## 📚 PV와 PVC 기본 개념

### 스토리지 레이어 아키텍처

```
┌─────────────────┐
│   Application   │ ← Pod에서 마운트 경로 사용
├─────────────────┤
│      PVC        │ ← 스토리지 요청 (클레임)
├─────────────────┤
│       PV        │ ← 실제 스토리지 리소스
├─────────────────┤
│ Storage Backend │ ← 물리적 스토리지 (NFS, Ceph, etc.)
└─────────────────┘
```

### PV 생명주기

```
Available → Bound → Released → (Recycled/Deleted/Failed)
```

## 🔧 PV 설정과 관리

### Static PV 생성

```yaml
# nfs-pv-example.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-web-data
  labels:
    type: nfs
    environment: production
    tier: frontend
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-storage
  mountOptions:
    - hard
    - nfsvers=4.1
    - rsize=1048576
    - wsize=1048576
    - timeo=600
    - retrans=2
  nfs:
    path: /export/web-data
    server: 192.168.1.100

---
# 로컬 스토리지 PV 예제
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-ssd-01
  labels:
    type: local
    performance: high
spec:
  capacity:
    storage: 500Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-ssd
  local:
    path: /mnt/ssd-storage
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-node-01

---
# Ceph RBD PV 예제
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ceph-rbd-pv-01
spec:
  capacity:
    storage: 200Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ceph-rbd
  rbd:
    monitors:
      - 192.168.1.10:6789
      - 192.168.1.11:6789
      - 192.168.1.12:6789
    pool: rbd
    image: kube-image-01
    user: admin
    secretRef:
      name: ceph-secret
    fsType: ext4
    readOnly: false
```

### PV 관리 스크립트

```bash
#!/bin/bash
# pv-management.sh

# PV 상태 모니터링
monitor_pv_status() {
    echo "=== PV Status Monitor ==="
    echo "Timestamp: $(date)"
    echo
    
    # 전체 PV 상태
    echo "=== All PVs ==="
    kubectl get pv -o wide
    echo
    
    # 상태별 PV 카운트
    echo "=== PV Status Summary ==="
    echo "Available: $(kubectl get pv --no-headers | grep Available | wc -l)"
    echo "Bound: $(kubectl get pv --no-headers | grep Bound | wc -l)"
    echo "Released: $(kubectl get pv --no-headers | grep Released | wc -l)"
    echo "Failed: $(kubectl get pv --no-headers | grep Failed | wc -l)"
    echo
    
    # 용량별 통계
    echo "=== Storage Capacity Summary ==="
    kubectl get pv --no-headers -o custom-columns=CAPACITY:.spec.capacity.storage | sort | uniq -c
    echo
    
    # 스토리지 클래스별 분포
    echo "=== PVs by Storage Class ==="
    kubectl get pv --no-headers -o custom-columns=SC:.spec.storageClassName | sort | uniq -c
}

# Released 상태 PV 정리
cleanup_released_pvs() {
    echo "=== Cleaning up Released PVs ==="
    
    released_pvs=$(kubectl get pv --no-headers | grep Released | awk '{print $1}')
    
    if [ -z "$released_pvs" ]; then
        echo "No Released PVs found"
        return
    fi
    
    echo "Found Released PVs:"
    echo "$released_pvs"
    echo
    
    read -p "Do you want to delete these PVs? (y/N): " -n 1 -r
    echo
    
    if [[ $REPLY =~ ^[Yy]$ ]]; then
        for pv in $released_pvs; do
            echo "Deleting PV: $pv"
            kubectl delete pv "$pv"
        done
    else
        echo "Cleanup cancelled"
    fi
}

# PV 백업 정보 생성
backup_pv_info() {
    local backup_dir="/var/backups/k8s-pv"
    local timestamp=$(date +%Y%m%d_%H%M%S)
    
    mkdir -p "$backup_dir"
    
    echo "Backing up PV information..."
    kubectl get pv -o yaml > "$backup_dir/pv-backup-$timestamp.yaml"
    kubectl get pvc --all-namespaces -o yaml > "$backup_dir/pvc-backup-$timestamp.yaml"
    
    echo "Backup completed: $backup_dir/pv-backup-$timestamp.yaml"
}

case "${1:-status}" in
    status)
        monitor_pv_status
        ;;
    cleanup)
        cleanup_released_pvs
        ;;
    backup)
        backup_pv_info
        ;;
    *)
        echo "Usage: $0 {status|cleanup|backup}"
        exit 1
        ;;
esac
```

## 📋 PVC 설정과 관리

### PVC 생성 예제

```yaml
# basic-pvc-examples.yaml
---
# 웹 애플리케이션용 PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-app-storage
  namespace: production
  labels:
    app: web-app
    tier: frontend
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: nfs-storage
  selector:
    matchLabels:
      type: nfs
      tier: frontend

---
# 데이터베이스용 고성능 PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
  namespace: database
  labels:
    app: mysql
    tier: database
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 200Gi
  storageClassName: local-ssd
  selector:
    matchLabels:
      type: local
      performance: high

---
# 로그 수집용 PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: log-storage
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Gi
  storageClassName: nfs-storage
```

### PVC 확장 (Volume Expansion)

```yaml
# 기존 PVC 확장
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: expandable-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi  # 기존 50Gi에서 100Gi로 확장
  storageClassName: expandable-storage-class
```

```bash
# PVC 확장 스크립트
#!/bin/bash
# expand-pvc.sh

PVC_NAME=$1
NAMESPACE=$2
NEW_SIZE=$3

if [ $# -ne 3 ]; then
    echo "Usage: $0 <pvc-name> <namespace> <new-size>"
    echo "Example: $0 mysql-data default 100Gi"
    exit 1
fi

echo "Expanding PVC $PVC_NAME in namespace $NAMESPACE to $NEW_SIZE"

# 현재 크기 확인
current_size=$(kubectl get pvc $PVC_NAME -n $NAMESPACE -o jsonpath='{.spec.resources.requests.storage}')
echo "Current size: $current_size"

# 스토리지 클래스 확장 가능 여부 확인
storage_class=$(kubectl get pvc $PVC_NAME -n $NAMESPACE -o jsonpath='{.spec.storageClassName}')
allow_expansion=$(kubectl get storageclass $storage_class -o jsonpath='{.allowVolumeExpansion}')

if [ "$allow_expansion" != "true" ]; then
    echo "Error: Storage class $storage_class does not allow volume expansion"
    exit 1
fi

# PVC 확장 실행
kubectl patch pvc $PVC_NAME -n $NAMESPACE -p "{\"spec\":{\"resources\":{\"requests\":{\"storage\":\"$NEW_SIZE\"}}}}"

echo "PVC expansion initiated. Monitoring progress..."

# 확장 상태 모니터링
timeout=300
while [ $timeout -gt 0 ]; do
    conditions=$(kubectl get pvc $PVC_NAME -n $NAMESPACE -o jsonpath='{.status.conditions[*].type}')
    if echo "$conditions" | grep -q "FileSystemResizePending"; then
        echo "File system resize pending... waiting for Pod restart"
    elif echo "$conditions" | grep -q "Resizing"; then
        echo "Volume resize in progress..."
    else
        new_size=$(kubectl get pvc $PVC_NAME -n $NAMESPACE -o jsonpath='{.status.capacity.storage}')
        echo "Expansion completed. New size: $new_size"
        break
    fi
    
    sleep 10
    timeout=$((timeout - 10))
done

if [ $timeout -le 0 ]; then
    echo "Warning: Expansion timeout. Check PVC status manually."
fi
```

## 🔄 Dynamic Provisioning

### StorageClass 와 Dynamic PV

```yaml
# dynamic-storage-classes.yaml
---
# NFS Dynamic Provisioner
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-dynamic
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: example.com/nfs
parameters:
  server: 192.168.1.100
  path: /export/dynamic
  archiveOnDelete: "false"
allowVolumeExpansion: true
volumeBindingMode: Immediate
reclaimPolicy: Delete

---
# Local SSD Dynamic Provisioner  
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-ssd-dynamic
provisioner: kubernetes.io/no-provisioner
parameters:
  type: "ssd"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: false
reclaimPolicy: Delete

---
# Ceph RBD Dynamic Provisioner
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd-dynamic
provisioner: rbd.csi.ceph.com
parameters:
  clusterID: ceph-cluster-id
  pool: rbd
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: ceph-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: ceph-csi-rbd
  csi.storage.k8s.io/controller-expand-secret-name: ceph-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: ceph-csi-rbd
  csi.storage.k8s.io/node-stage-secret-name: ceph-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: ceph-csi-rbd
allowVolumeExpansion: true
volumeBindingMode: Immediate
reclaimPolicy: Delete
```

### NFS Provisioner 배포

```yaml
# nfs-provisioner.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: example.com/nfs
            - name: NFS_SERVER
              value: 192.168.1.100
            - name: NFS_PATH
              value: /export/dynamic
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.1.100
            path: /export/dynamic

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: kube-system

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io

---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: kube-system
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: kube-system
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

## 📊 스토리지 모니터링과 관리

### PVC 사용량 모니터링

```bash
#!/bin/bash
# storage-monitoring.sh

echo "=== Storage Usage Monitoring ==="
echo "Timestamp: $(date)"
echo

# PVC 사용량 상세 정보
echo "=== PVC Usage Details ==="
kubectl get pvc --all-namespaces -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,STATUS:.status.phase,VOLUME:.spec.volumeName,CAPACITY:.status.capacity.storage,STORAGECLASS:.spec.storageClassName"
echo

# 네임스페이스별 스토리지 사용량 요약
echo "=== Storage Usage by Namespace ==="
for ns in $(kubectl get namespaces -o name | cut -d/ -f2); do
    pvc_count=$(kubectl get pvc -n $ns --no-headers 2>/dev/null | wc -l)
    if [ $pvc_count -gt 0 ]; then
        echo "Namespace: $ns"
        kubectl get pvc -n $ns --no-headers -o custom-columns="NAME:.metadata.name,CAPACITY:.status.capacity.storage"
        echo
    fi
done

# 스토리지 클래스별 사용량
echo "=== Usage by Storage Class ==="
kubectl get pvc --all-namespaces --no-headers -o custom-columns="SC:.spec.storageClassName,CAPACITY:.status.capacity.storage" | sort | awk '
{
    sc = $1
    capacity = $2
    if (sc in total) {
        total[sc] = total[sc] " + " capacity
    } else {
        total[sc] = capacity
        count[sc] = 0
    }
    count[sc]++
}
END {
    for (sc in total) {
        printf "%-20s: %d PVCs\n", sc, count[sc]
    }
}'

# 문제가 있는 PVC 확인
echo "=== Problematic PVCs ==="
kubectl get pvc --all-namespaces --field-selector status.phase!=Bound 2>/dev/null || echo "All PVCs are Bound"
```

### 스토리지 정리 및 최적화

```bash
#!/bin/bash
# storage-cleanup.sh

# 사용하지 않는 PV 찾기
find_unused_pvs() {
    echo "=== Finding Unused PVs ==="
    
    # Available 상태의 PV들
    available_pvs=$(kubectl get pv --no-headers | grep Available | awk '{print $1}')
    if [ -n "$available_pvs" ]; then
        echo "Available PVs (not bound to any PVC):"
        echo "$available_pvs"
        echo
    fi
    
    # Released 상태의 PV들  
    released_pvs=$(kubectl get pv --no-headers | grep Released | awk '{print $1}')
    if [ -n "$released_pvs" ]; then
        echo "Released PVs (previously bound, now released):"
        echo "$released_pvs"
        echo
    fi
}

# 오래된 완료된 Pod의 PVC 확인
find_stale_pvcs() {
    echo "=== Finding Potentially Stale PVCs ==="
    
    # 완료된 Job/Pod와 연결된 PVC 찾기
    for ns in $(kubectl get namespaces -o name | cut -d/ -f2); do
        completed_pods=$(kubectl get pods -n $ns --no-headers | grep -E 'Completed|Succeeded' | awk '{print $1}')
        
        for pod in $completed_pods; do
            pvcs=$(kubectl get pod $pod -n $ns -o jsonpath='{.spec.volumes[*].persistentVolumeClaim.claimName}' 2>/dev/null)
            if [ -n "$pvcs" ]; then
                echo "Namespace $ns, Completed Pod $pod uses PVCs: $pvcs"
            fi
        done
    done
}

# 임시 스토리지 정리
cleanup_temp_storage() {
    echo "=== Cleaning up temporary storage ==="
    
    # 각 노드에서 임시 스토리지 정리
    for node in $(kubectl get nodes -o name | cut -d/ -f2); do
        echo "Cleaning node: $node"
        
        # SSH로 접근 가능한 경우 정리 실행
        ssh $node "
            # containerd 이미지 정리
            crictl rmi --prune
            
            # 사용하지 않는 볼륨 정리
            find /var/lib/kubelet/pods -name 'volumes' -type d -empty -delete 2>/dev/null || true
            
            # 임시 파일 정리
            find /tmp -name 'kubectl-*' -mtime +7 -delete 2>/dev/null || true
            
            echo 'Node $node cleanup completed'
        " 2>/dev/null || echo "SSH access not available for node $node"
    done
}

case "${1:-check}" in
    check)
        find_unused_pvs
        find_stale_pvcs
        ;;
    cleanup)
        cleanup_temp_storage
        ;;
    *)
        echo "Usage: $0 {check|cleanup}"
        exit 1
        ;;
esac
```

## 🔒 PV/PVC 보안 관리

### 접근 권한 제어

```yaml
# pvc-rbac.yaml
---
# PVC 생성 권한을 가진 Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pvc-manager
rules:
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list", "watch"]

---
# 읽기 전용 PVC 접근 권한
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pvc-viewer
rules:
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list", "watch"]

---
# RoleBinding 예제
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pvc-managers
  namespace: production
subjects:
- kind: User
  name: storage-admin
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: storage-operator
  namespace: production
roleRef:
  kind: Role
  name: pvc-manager
  apiGroup: rbac.authorization.k8s.io
```

### 스토리지 정책 적용

```yaml
# storage-policies.yaml
---
# ResourceQuota로 스토리지 사용량 제한
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: development
spec:
  hard:
    requests.storage: "500Gi"
    persistentvolumeclaims: "10"

---
# LimitRange로 PVC 크기 제한
apiVersion: v1
kind: LimitRange
metadata:
  name: pvc-limit-range
  namespace: development
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: "100Gi"
    min:
      storage: "1Gi"
    default:
      storage: "10Gi"
    defaultRequest:
      storage: "5Gi"
```

## 📋 운영 베스트 프랙티스

### 1. PV/PVC 명명 규칙

```bash
# 권장 명명 규칙
# PV: <스토리지타입>-<환경>-<용도>-<일련번호>
nfs-prod-web-data-001
local-ssd-prod-db-mysql-001
ceph-rbd-staging-logs-001

# PVC: <애플리케이션>-<데이터타입>-<환경>
wordpress-data-prod
mysql-data-staging
redis-cache-dev
```

### 2. 레이블링 전략

```yaml
# 표준 레이블 예제
metadata:
  labels:
    # 애플리케이션 정보
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-primary
    app.kubernetes.io/component: database
    
    # 스토리지 관련
    storage.company.com/type: database
    storage.company.com/performance: high
    storage.company.com/backup-policy: daily
    
    # 환경 정보
    environment: production
    tier: data
```

### 3. 백업 및 복구 전략

```bash
# PV 백업 스크립트
#!/bin/bash
backup_pv_data() {
    local pv_name=$1
    local backup_path="/backups/pv-data"
    local timestamp=$(date +%Y%m%d_%H%M%S)
    
    # PV 정보 획득
    local mount_path=$(kubectl get pv $pv_name -o jsonpath='{.spec.hostPath.path}')
    local node=$(kubectl get pv $pv_name -o jsonpath='{.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]}')
    
    # 데이터 백업
    ssh $node "tar czf $backup_path/$pv_name-$timestamp.tar.gz -C $mount_path ."
    
    echo "Backup completed: $backup_path/$pv_name-$timestamp.tar.gz"
}
```

### 4. 모니터링 설정

```yaml
# PrometheusRule for PV/PVC monitoring
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: storage-monitoring
  namespace: monitoring
spec:
  groups:
  - name: storage.rules
    rules:
    - alert: PVCPendingForTooLong
      expr: kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1
      for: 5m
      annotations:
        summary: "PVC {{ $labels.persistentvolumeclaim }} is pending for too long"
        
    - alert: PVDiskSpaceUsageHigh
      expr: (kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) > 0.9
      for: 2m
      annotations:
        summary: "PV disk usage is above 90%"
```

---

> 💡 **실전 경험**: PV/PVC 관리에서 가장 중요한 것은 consistent한 명명 규칙과 레이블링입니다. 초기에 잘 설계해두면 나중에 자동화 스크립트 작성과 모니터링이 훨씬 쉬워집니다.

태그: #storage #pv #pvc #management #onprem #backup
