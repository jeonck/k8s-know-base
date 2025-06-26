# StatefulSet 운영

## 🎯 개요

StatefulSet은 상태를 가진 애플리케이션(데이터베이스, 메시지 큐, 분산 시스템 등)을 쿠버네티스에서 운영하기 위한 워크로드 컨트롤러입니다. 온프렘 환경에서의 StatefulSet 설계, 배포, 운영 방법을 실전 중심으로 다룹니다.

## 📊 StatefulSet vs Deployment

### 주요 차이점

| 특성 | Deployment | StatefulSet |
|------|------------|-------------|
| **Pod 이름** | 랜덤 | 순차적 (app-0, app-1, app-2) |
| **Pod 순서** | 동시 생성/삭제 | 순차적 생성/삭제 |
| **네트워크 ID** | 임시 | 안정적 (Headless Service) |
| **스토리지** | 공유 가능 | 개별 PVC |
| **업데이트** | Rolling/Recreate | RollingUpdate/OnDelete |
| **용도** | 무상태 앱 | 상태 유지 앱 |

### StatefulSet이 필요한 경우

```yaml
상태를 가진 애플리케이션:
  - 데이터베이스 (MySQL, PostgreSQL, MongoDB)
  - 메시지 큐 (Kafka, RabbitMQ)
  - 분산 시스템 (Elasticsearch, Cassandra)
  - 캐시 시스템 (Redis Cluster)

요구사항:
  - 안정적인 네트워크 ID 필요
  - 순차적인 시작/종료 필요
  - 개별 영구 스토리지 필요
  - 클러스터 멤버십 관리 필요
```

## 🗄️ 데이터베이스 StatefulSet

### MySQL 클러스터 구성

```yaml
# mysql-statefulset.yaml
---
# Headless Service (안정적인 네트워크 ID 제공)
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
  namespace: database
  labels:
    app: mysql
spec:
  clusterIP: None  # Headless Service
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql

---
# MySQL StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: database
spec:
  serviceName: mysql-headless
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      # 서버 ID 설정 (Pod 순서 기반)
      - name: init-mysql
        image: mysql:8.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Pod 순서에서 서버 ID 추출 (mysql-0 -> 100, mysql-1 -> 101)
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # 마스터/슬레이브 설정
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      
      # 기존 데이터 클론 (슬레이브용)
      - name: clone-mysql
        image: percona/percona-xtrabackup:8.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          # 이전 Pod에서 데이터 클론
          ncat --recv-only mysql-$(($ordinal-1)).mysql-headless 3307 | xbstream -x -C /var/lib/mysql
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: "1"
        ports:
        - name: mysql
          containerPort: 3306
        # 볼륨 마운트
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        # 리소스 제한
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
        # 헬스체크
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
      
      # 백업 사이드카
      - name: xtrabackup
        image: percona/percona-xtrabackup:8.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql
          if [[ -f xtrabackup_slave_info && "x$(<xtrabackup_slave_info)" != "x" ]]; then
            cat xtrabackup_slave_info | sed -E 's/;$//g' > change_master_to.sql.in
            rm -f xtrabackup_slave_info xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm -f xtrabackup_binlog_info xtrabackup_slave_info
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi
          if [[ -f change_master_to.sql.in ]]; then
            echo "Waiting for mysqld to be ready..."
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done
            echo "Initializing replication from clone position"
            mysql -h 127.0.0.1 <<EOF
          $(<change_master_to.sql.in),
            MASTER_HOST='mysql-0.mysql-headless',
            MASTER_USER='root',
            MASTER_PASSWORD='',
            MASTER_CONNECT_RETRY=10;
          START SLAVE;
          EOF
          fi
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 200m
            memory: 200Mi
      
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql-config
  
  # PVC 템플릿 (각 Pod마다 개별 PVC 생성)
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 100Gi

---
# MySQL 설정 ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: database
data:
  master.cnf: |
    [mysqld]
    log-bin=mysql-bin
    log-slave-updates=1
    binlog-format=ROW
    
  slave.cnf: |
    [mysqld]
    super-read-only=1
    log-slave-updates=1
    read-only=1
```

### MongoDB ReplicaSet

```yaml
# mongodb-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: database
spec:
  serviceName: mongodb-headless
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:6.0
        command:
        - mongod
        - "--replSet"
        - "rs0"
        - "--bind_ip_all"
        - "--wiredTigerCacheSizeGB"
        - "1"
        ports:
        - containerPort: 27017
          name: mongodb
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /data/db
        # 헬스체크
        livenessProbe:
          exec:
            command:
            - mongosh
            - --eval
            - "db.adminCommand('ping')"
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - mongosh
            - --eval
            - "db.adminCommand('ping')"
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "2Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "1000m"
      
      # MongoDB ReplicaSet 초기화
      initContainers:
      - name: setup-replica-set
        image: mongo:6.0
        command:
        - bash
        - -c
        - |
          if [[ "${HOSTNAME}" == "mongodb-0" ]]; then
            echo "Waiting for MongoDB to start..."
            until mongosh --host mongodb-0.mongodb-headless:27017 --eval "print('waited for connection')"
            do
              sleep 2
            done
            echo "Initializing replica set..."
            mongosh --host mongodb-0.mongodb-headless:27017 <<EOF
            rs.initiate({
              _id: "rs0",
              members: [
                {_id: 0, host: "mongodb-0.mongodb-headless:27017", priority: 2},
                {_id: 1, host: "mongodb-1.mongodb-headless:27017", priority: 1},
                {_id: 2, host: "mongodb-2.mongodb-headless:27017", priority: 1}
              ]
            })
          EOF
          fi
  
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 50Gi

---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-headless
  namespace: database
spec:
  clusterIP: None
  selector:
    app: mongodb
  ports:
  - port: 27017
    name: mongodb
```

## 🔄 StatefulSet 업데이트 전략

### RollingUpdate 설정

```yaml
# statefulset-update-strategy.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-cluster
  namespace: production
spec:
  # 업데이트 전략
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1      # 동시에 업데이트할 Pod 수
      partition: 0           # 업데이트 시작할 Pod 순서 (역순)
  
  # 기본 설정
  serviceName: web-cluster-headless
  replicas: 5
  selector:
    matchLabels:
      app: web-cluster
  template:
    metadata:
      labels:
        app: web-cluster
    spec:
      containers:
      - name: web-app
        image: myapp:v2.0
        # Pod 간 의존성이 있는 경우 순차 시작 보장
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        # Graceful shutdown 설정
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                # 클러스터에서 자신을 제거
                curl -X POST http://localhost:8080/cluster/leave
                sleep 10
      # 종료 대기 시간
      terminationGracePeriodSeconds: 60
```

### Canary 업데이트 (Partition 활용)

```bash
#!/bin/bash
# statefulset-canary-update.sh

NAMESPACE="production"
STATEFULSET="web-cluster"
NEW_IMAGE="myapp:v2.0"
TOTAL_REPLICAS=5

echo "Starting StatefulSet canary update..."

# 1. 최신 Pod 하나만 업데이트 (partition 설정)
kubectl patch statefulset $STATEFULSET -n $NAMESPACE -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":'$((TOTAL_REPLICAS-1))'}}}}'

# 2. 이미지 업데이트
kubectl set image statefulset/$STATEFULSET -n $NAMESPACE web-app=$NEW_IMAGE

# 3. 마지막 Pod(가장 높은 번호) 업데이트 대기
echo "Updating pod $STATEFULSET-$((TOTAL_REPLICAS-1))..."
kubectl rollout status statefulset/$STATEFULSET -n $NAMESPACE --timeout=300s

# 4. Canary Pod 검증
echo "Validating canary pod..."
CANARY_POD="$STATEFULSET-$((TOTAL_REPLICAS-1))"

# 헬스체크
if kubectl exec -n $NAMESPACE $CANARY_POD -- curl -f http://localhost:8080/health > /dev/null 2>&1; then
    echo "✅ Canary pod health check passed"
else
    echo "❌ Canary pod health check failed"
    exit 1
fi

# 5. 사용자 승인 후 전체 업데이트
read -p "Canary validation successful. Continue with full update? (y/N): " -n 1 -r
echo

if [[ $REPLY =~ ^[Yy]$ ]]; then
    echo "Starting full StatefulSet update..."
    kubectl patch statefulset $STATEFULSET -n $NAMESPACE -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
    kubectl rollout status statefulset/$STATEFULSET -n $NAMESPACE --timeout=600s
    echo "✅ StatefulSet update completed"
else
    echo "Update cancelled. Rolling back canary pod..."
    # Partition을 원래대로 되돌리고 이전 이미지로 복원
    kubectl patch statefulset $STATEFULSET -n $NAMESPACE -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":'$TOTAL_REPLICAS'}}}}'
fi
```

## 🔧 StatefulSet 스케일링

### 수평 스케일링

```bash
#!/bin/bash
# statefulset-scaling.sh

NAMESPACE="database"
STATEFULSET="mysql"
TARGET_REPLICAS=$1

if [ -z "$TARGET_REPLICAS" ]; then
    echo "Usage: $0 <target-replicas>"
    exit 1
fi

CURRENT_REPLICAS=$(kubectl get statefulset $STATEFULSET -n $NAMESPACE -o jsonpath='{.spec.replicas}')

echo "Scaling $STATEFULSET from $CURRENT_REPLICAS to $TARGET_REPLICAS replicas"

if [ $TARGET_REPLICAS -gt $CURRENT_REPLICAS ]; then
    echo "Scaling UP..."
    kubectl scale statefulset $STATEFULSET -n $NAMESPACE --replicas=$TARGET_REPLICAS
    
    # 새 Pod들이 준비될 때까지 대기
    for ((i=$CURRENT_REPLICAS; i<$TARGET_REPLICAS; i++)); do
        echo "Waiting for $STATEFULSET-$i to be ready..."
        kubectl wait --for=condition=ready pod/$STATEFULSET-$i -n $NAMESPACE --timeout=300s
        
        # 데이터베이스 클러스터의 경우 새 멤버 추가 작업
        if [ "$STATEFULSET" = "mysql" ]; then
            echo "Adding $STATEFULSET-$i to MySQL cluster..."
            # MySQL 클러스터 설정 작업
        fi
    done
    
elif [ $TARGET_REPLICAS -lt $CURRENT_REPLICAS ]; then
    echo "Scaling DOWN..."
    
    # 스케일 다운 전 데이터 안전성 확인
    for ((i=$((TARGET_REPLICAS)); i<$CURRENT_REPLICAS; i++)); do
        echo "Preparing to remove $STATEFULSET-$i..."
        
        # 데이터베이스의 경우 데이터 마이그레이션
        if [ "$STATEFULSET" = "mysql" ]; then
            echo "Ensuring data safety for $STATEFULSET-$i..."
            # 복제 상태 확인, 데이터 백업 등
        fi
    done
    
    kubectl scale statefulset $STATEFULSET -n $NAMESPACE --replicas=$TARGET_REPLICAS
    
    echo "⚠️  Remember to manually clean up PVCs if needed:"
    for ((i=$TARGET_REPLICAS; i<$CURRENT_REPLICAS; i++)); do
        echo "  kubectl delete pvc data-$STATEFULSET-$i -n $NAMESPACE"
    done
fi

echo "✅ Scaling completed"
```

### 수직 스케일링 (리소스 조정)

```yaml
# statefulset-vertical-scaling.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: database
spec:
  template:
    spec:
      containers:
      - name: mongodb
        image: mongo:6.0
        # 리소스 요구사항 증가
        resources:
          requests:
            memory: "4Gi"    # 2Gi → 4Gi
            cpu: "1000m"     # 500m → 1000m
          limits:
            memory: "8Gi"    # 4Gi → 8Gi
            cpu: "2000m"     # 1000m → 2000m
        # MongoDB 설정도 함께 조정
        command:
        - mongod
        - "--wiredTigerCacheSizeGB"
        - "3"              # 캐시 크기 증가
```

## 💾 StatefulSet 백업과 복원

### MySQL 백업 전략

```bash
#!/bin/bash
# mysql-backup.sh

NAMESPACE="database"
BACKUP_DIR="/var/backups/mysql"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p "$BACKUP_DIR"

# 각 MySQL Pod 백업
for i in {0..2}; do
    POD_NAME="mysql-$i"
    BACKUP_FILE="$BACKUP_DIR/mysql-$i-backup-$DATE.sql"
    
    echo "Backing up $POD_NAME..."
    
    kubectl exec -n $NAMESPACE $POD_NAME -- mysqldump \
        --all-databases \
        --single-transaction \
        --routines \
        --triggers \
        --master-data=2 > "$BACKUP_FILE"
    
    if [ $? -eq 0 ]; then
        echo "✅ Backup completed: $BACKUP_FILE"
        gzip "$BACKUP_FILE"
    else
        echo "❌ Backup failed for $POD_NAME"
    fi
done

# 오래된 백업 정리 (30일 이상)
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +30 -delete

echo "MySQL backup completed"
```

### StatefulSet 재해 복구

```bash
#!/bin/bash
# statefulset-disaster-recovery.sh

NAMESPACE="database"
STATEFULSET="mysql"
BACKUP_FILE="/var/backups/mysql/mysql-0-backup-20241201_020000.sql.gz"

echo "Starting StatefulSet disaster recovery..."

# 1. StatefulSet 스케일 다운
echo "Scaling down StatefulSet..."
kubectl scale statefulset $STATEFULSET -n $NAMESPACE --replicas=0

# 2. PVC 데이터 백업 (기존 데이터 보존)
echo "Backing up existing PVC data..."
for i in {0..2}; do
    kubectl create job backup-pvc-$i -n $NAMESPACE --image=busybox -- \
        tar czf /backup/pvc-data-$i-$(date +%Y%m%d).tar.gz -C /data .
done

# 3. PVC 초기화 (필요한 경우)
read -p "Delete existing PVC data? (y/N): " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    for i in {0..2}; do
        kubectl delete pvc data-$STATEFULSET-$i -n $NAMESPACE
    done
fi

# 4. StatefulSet 재시작
echo "Scaling up StatefulSet..."
kubectl scale statefulset $STATEFULSET -n $NAMESPACE --replicas=3

# 5. 첫 번째 Pod 복원
echo "Waiting for mysql-0 to be ready..."
kubectl wait --for=condition=ready pod/mysql-0 -n $NAMESPACE --timeout=300s

echo "Restoring data to mysql-0..."
zcat "$BACKUP_FILE" | kubectl exec -i -n $NAMESPACE mysql-0 -- mysql

# 6. 복제 재설정
echo "Setting up replication..."
kubectl exec -n $NAMESPACE mysql-1 -- mysql -e "
STOP SLAVE;
RESET SLAVE ALL;
CHANGE MASTER TO 
  MASTER_HOST='mysql-0.mysql-headless',
  MASTER_USER='root',
  MASTER_PASSWORD='';
START SLAVE;"

kubectl exec -n $NAMESPACE mysql-2 -- mysql -e "
STOP SLAVE;
RESET SLAVE ALL;
CHANGE MASTER TO 
  MASTER_HOST='mysql-0.mysql-headless',
  MASTER_USER='root',
  MASTER_PASSWORD='';
START SLAVE;"

echo "✅ Disaster recovery completed"
```

## 📊 StatefulSet 모니터링

### 상태 모니터링 메트릭

```yaml
# statefulset-monitoring.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: statefulset-alerts
  namespace: monitoring
spec:
  groups:
  - name: statefulset.rules
    rules:
    - alert: StatefulSetReplicasMismatch
      expr: kube_statefulset_status_replicas_ready != kube_statefulset_spec_replicas
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "StatefulSet {{ $labels.statefulset }} replica mismatch"
        description: "StatefulSet {{ $labels.statefulset }} has {{ $value }} ready replicas but wants {{ $labels.spec_replicas }}"
        
    - alert: StatefulSetUpdateNotProgressing
      expr: kube_statefulset_status_observed_generation != kube_statefulset_metadata_generation
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "StatefulSet {{ $labels.statefulset }} update not progressing"
        
    - alert: StatefulSetPodNotReady
      expr: kube_pod_status_ready{condition="false"} * on(pod) group_left(statefulset) kube_pod_info{created_by_kind="StatefulSet"}
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "StatefulSet pod {{ $labels.pod }} not ready"
```

## 🔧 StatefulSet 트러블슈팅

### 일반적인 문제들

#### 1. Pod가 Pending 상태
```bash
# PVC 상태 확인
kubectl get pvc -n $NAMESPACE

# 스토리지 클래스 확인
kubectl get storageclass

# 이벤트 확인
kubectl get events -n $NAMESPACE --sort-by='.lastTimestamp'
```

#### 2. Pod 시작 순서 문제
```bash
# Pod 의존성 확인
kubectl describe pod $POD_NAME -n $NAMESPACE

# Init Container 로그 확인
kubectl logs $POD_NAME -c init-container -n $NAMESPACE

# readinessProbe 설정 확인
kubectl get pod $POD_NAME -n $NAMESPACE -o yaml | grep -A 10 readinessProbe
```

#### 3. 데이터 손실 문제
```bash
# PV 상태 확인
kubectl get pv

# 백업 존재 여부 확인
ls -la /var/backups/

# 데이터 복구 절차 실행
./statefulset-disaster-recovery.sh
```

---

> 💡 **실전 경험**: StatefulSet은 상태를 가진 애플리케이션의 특성을 이해하고 운영해야 합니다. 특히 데이터 백업, 클러스터 멤버십 관리, 순차적 업데이트에 신경 써야 하며, 스케일링 시에는 데이터 안전성을 최우선으로 고려해야 합니다.

태그: #statefulset #database #mysql #mongodb #scaling #backup #recovery