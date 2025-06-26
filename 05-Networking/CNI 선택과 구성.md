# CNI 선택과 구성

## 🎯 개요

쿠버네티스 클러스터의 네트워킹을 담당하는 CNI(Container Network Interface) 플러그인 선택과 구성 방법을 다룹니다. 온프렘 환경에서 각 CNI의 특징과 최적 구성을 제시합니다.

## 🌐 CNI 비교와 선택 기준

### 주요 CNI 플러그인 비교

| CNI | 성능 | 복잡도 | 네트워크 정책 | 온프렘 적합성 | 특징 |
|-----|------|--------|---------------|---------------|------|
| **Calico** | ⭐⭐⭐⭐ | 중간 | ✅ 강력 | ⭐⭐⭐⭐⭐ | BGP, IPIP, VXLAN 지원 |
| **Flannel** | ⭐⭐⭐ | 낮음 | ❌ 기본 없음 | ⭐⭐⭐⭐ | 간단한 설정, 안정성 |
| **Weave** | ⭐⭐ | 중간 | ✅ 있음 | ⭐⭐⭐ | 자동 탐지, 암호화 |
| **Cilium** | ⭐⭐⭐⭐⭐ | 높음 | ✅ 강력 | ⭐⭐⭐ | eBPF 기반, 고성능 |
| **Antrea** | ⭐⭐⭐⭐ | 중간 | ✅ 있음 | ⭐⭐⭐⭐ | OVS 기반, VMware 지원 |

### 온프렘 환경별 권장사항

```yaml
소규모 클러스터 (1-10 노드):
  권장: Flannel
  이유: 간단한 설정, 안정성, 낮은 학습 곡선

중규모 클러스터 (10-100 노드):
  권장: Calico
  이유: 네트워크 정책, 확장성, BGP 지원

대규모 클러스터 (100+ 노드):
  권장: Cilium 또는 Calico
  이유: 고성능, 고급 네트워킹 기능

보안 중시 환경:
  권장: Calico 또는 Cilium
  이유: 강력한 네트워크 정책, 보안 기능

기존 네트워크 인프라 통합:
  권장: Calico (BGP 모드)
  이유: 기존 라우터와 BGP 연동 가능
```

## 🐧 Calico 구성

### Calico 설치 (BGP 모드)

```bash
# Calico Operator 설치
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/tigera-operator.yaml

# Calico 사용자 정의 리소스 생성
cat <<EOF | kubectl create -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # BGP 모드 사용 (온프렘 권장)
  calicoNetwork:
    bgp: Enabled
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: None  # BGP 모드에서는 encapsulation 없음
      natOutgoing: Enabled
      nodeSelector: all()
  # 노드별 설정
  nodeAddressAutodetection:
    interface: "eth0"  # 실제 인터페이스명으로 변경
  flexVolumePath: /usr/libexec/kubernetes/kubelet-plugins/volume/exec/
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF
```

### Calico BGP 피어링 설정

```yaml
# bgp-configuration.yaml
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: true  # 노드 간 full mesh
  asNumber: 64512              # 클러스터 AS 번호
  serviceClusterIPs:           # Service CIDR BGP 광고
  - cidr: 10.96.0.0/12
  serviceExternalIPs:          # External IP BGP 광고
  - cidr: 192.168.1.0/24

---
# 외부 라우터와 BGP 피어링
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: rack1-router
spec:
  peerIP: 192.168.1.1    # 라우터 IP
  asNumber: 65001        # 라우터 AS 번호
  # 특정 노드에만 적용 (선택사항)
  nodeSelector: rack == "rack1"
```

### Calico IPIP/VXLAN 모드 (cross-subnet)

```yaml
# calico-ipip-config.yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    bgp: Enabled
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: IPIP          # IPIP 터널링
      natOutgoing: Enabled
      ipipMode: CrossSubnet        # 다른 서브넷 간에만 IPIP 사용
      nodeSelector: all()
```

## 🏗️ Flannel 구성

### Flannel 설치 (VXLAN 모드)

```bash
# Flannel 설치
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/v0.22.0/Documentation/kube-flannel.yml

# Flannel 설정 확인
kubectl get pods -n kube-flannel
kubectl get configmap -n kube-flannel kube-flannel-cfg -o yaml
```

### Flannel 커스텀 설정

```yaml
# flannel-custom-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan",
        "VNI": 1,
        "Port": 8472
      }
    }

---
# Flannel DaemonSet 설정
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni-plugin
        image: docker.io/flannel/flannel-cni-plugin:v1.1.0
        command:
        - cp
        args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        volumeMounts:
        - name: cni-plugin
          mountPath: /opt/cni/bin
      - name: install-cni
        image: docker.io/flannel/flannel:v0.22.0
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: docker.io/flannel/flannel:v0.22.0
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=eth0  # 실제 인터페이스명
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: EVENT_QUEUE_DEPTH
          value: "5000"
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
        - name: xtables-lock
          mountPath: /run/xtables.lock
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni-plugin
        hostPath:
          path: /opt/cni/bin
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
```

## 🕷️ Cilium 구성

### Cilium 설치 (Helm)

```bash
# Helm으로 Cilium 설치
helm repo add cilium https://helm.cilium.io/
helm repo update

# Cilium 설치 (기본 설정)
helm install cilium cilium/cilium \
  --version 1.14.0 \
  --namespace kube-system \
  --set operator.replicas=1 \
  --set ipam.mode=kubernetes \
  --set kubeProxyReplacement=partial \
  --set hostServices.enabled=false \
  --set externalIPs.enabled=true \
  --set nodePort.enabled=true \
  --set hostPort.enabled=true \
  --set image.pullPolicy=IfNotPresent \
  --set ipv4.enabled=true \
  --set ipv6.enabled=false
```

### Cilium 고급 설정

```yaml
# cilium-values.yaml
# helm install cilium cilium/cilium -f cilium-values.yaml

# 기본 설정
cluster:
  name: onprem-cluster
  id: 1

# IPAM 설정
ipam:
  mode: kubernetes

# 네트워킹 설정
ipv4:
  enabled: true
ipv6:
  enabled: false

# 터널링 설정 (VXLAN)
tunnelProtocol: "vxlan"
autoDirectNodeRoutes: true

# kube-proxy 대체 설정
kubeProxyReplacement: "partial"
k8sServiceHost: "10.1.1.100"  # API 서버 LB
k8sServicePort: "6443"

# eBPF 기능 활성화
bpf:
  clockProbe: true
  preallocateMaps: false
  lbMapMax: 65536
  policyMapMax: 16384
  monitorAggregation: medium

# Hubble (관찰성)
hubble:
  enabled: true
  metrics:
    enabled:
    - dns:query;ignoreAAAA
    - drop
    - tcp
    - flow
    - icmp
    - http
  relay:
    enabled: true
  ui:
    enabled: true

# 운영자 설정
operator:
  replicas: 1
  resources:
    limits:
      cpu: 1000m
      memory: 1Gi
    requests:
      cpu: 100m
      memory: 128Mi

# 에이전트 설정
resources:
  limits:
    cpu: 4000m
    memory: 4Gi
  requests:
    cpu: 100m
    memory: 512Mi

# 보안 설정
encryption:
  enabled: false  # WireGuard 암호화 (필요시 활성화)
  type: wireguard

# 네트워크 정책
policyEnforcementMode: "default"
```

## 🔒 네트워크 정책 구성

### Calico 네트워크 정책

```yaml
# calico-network-policies.yaml
---
# 기본 거부 정책
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
# 웹 애플리케이션 정책
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
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to: []  # DNS 허용
    ports:
    - protocol: UDP
      port: 53

---
# Calico GlobalNetworkPolicy (클러스터 전역)
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: deny-node-access
spec:
  # 노드 포트로의 직접 접근 차단
  order: 1000
  selector: all()
  types:
  - Ingress
  ingress:
  - action: Deny
    destination:
      ports:
      - 22    # SSH
      - 6443  # API Server
      - 2379  # etcd
      - 2380  # etcd peer
    source:
      notSelector: role == "admin"
```

### Cilium 네트워크 정책 (L7)

```yaml
# cilium-l7-policy.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: l7-http-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: web-api
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/v1/.*"
        - method: "POST"
          path: "/api/v1/users"
          headers:
          - "Content-Type: application/json"
```

## 📊 CNI 성능 모니터링

### 네트워크 메트릭 수집

```yaml
# cni-monitoring.yaml
---
# Calico 메트릭
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: calico-node
  namespace: monitoring
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  endpoints:
  - port: http-metrics
    interval: 30s

---
# Cilium 메트릭
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: cilium-agent
  namespace: monitoring
spec:
  selector:
    matchLabels:
      k8s-app: cilium
  endpoints:
  - port: prometheus
    interval: 30s
    path: /metrics
```

### 네트워크 성능 테스트

```bash
#!/bin/bash
# network-performance-test.sh

echo "=== Network Performance Test ==="

# Pod 간 대역폭 테스트
kubectl run netperf-server --image=networkstatic/netperf --port=12865 --restart=Never -- netserver -D
kubectl run netperf-client --image=networkstatic/netperf --restart=Never --rm -it -- netperf -H netperf-server -t TCP_STREAM

# 지연 시간 테스트
kubectl run ping-test --image=busybox --restart=Never --rm -it -- ping -c 10 netperf-server

# DNS 성능 테스트
kubectl run dns-test --image=busybox --restart=Never --rm -it -- nslookup kubernetes.default.svc.cluster.local

# 정리
kubectl delete pod netperf-server
```

## 🔧 CNI 트러블슈팅

### 일반적인 네트워크 문제

```bash
# CNI 플러그인 상태 확인
kubectl get pods -n kube-system | grep -E "calico|flannel|cilium"

# CNI 설정 확인
ls -la /etc/cni/net.d/
cat /etc/cni/net.d/10-calico.conflist

# 네트워크 인터페이스 확인
ip addr show
ip route show

# iptables 규칙 확인 (Calico/Flannel)
iptables -L -n | grep KUBE
iptables -L -n -t nat | grep KUBE

# 컨테이너 네트워크 네임스페이스 확인
docker exec <container-id> ip addr show
```

### Calico 트러블슈팅

```bash
# Calico 노드 상태 확인
kubectl exec -n calico-system calico-node-xxx -- calicoctl node status

# BGP 피어 상태 확인
kubectl exec -n calico-system calico-node-xxx -- calicoctl node status
kubectl exec -n calico-system calico-node-xxx -- bird -s /var/run/calico/bird.ctl

# 라우팅 테이블 확인
kubectl exec -n calico-system calico-node-xxx -- ip route show

# Calico 로그 확인
kubectl logs -n calico-system calico-node-xxx -c calico-node
```

### Cilium 트러블슈팅

```bash
# Cilium 상태 확인
kubectl exec -n kube-system cilium-xxx -- cilium status

# Cilium 연결성 테스트
kubectl exec -n kube-system cilium-xxx -- cilium connectivity test

# eBPF 맵 확인
kubectl exec -n kube-system cilium-xxx -- cilium bpf lb list
kubectl exec -n kube-system cilium-xxx -- cilium bpf endpoint list

# Hubble 관찰
kubectl exec -n kube-system cilium-xxx -- hubble observe --follow
```

## 📋 CNI 선택 가이드

### 환경별 권장사항

```yaml
개발/테스트 환경:
  권장: Flannel
  이유: 간단한 설정, 빠른 구축
  
소규모 운영 환경:
  권장: Calico (IPIP 모드)
  이유: 네트워크 정책, 안정성
  
대규모 운영 환경:
  권장: Cilium 또는 Calico (BGP 모드)
  이유: 고성능, 확장성
  
보안 중요 환경:
  권장: Cilium (eBPF) 또는 Calico
  이유: L7 정책, 고급 보안 기능
  
기존 네트워크 통합:
  권장: Calico (BGP 모드)
  이유: 라우터와 BGP 연동
```

### 성능 고려사항

```bash
# 대역폭 우선: Cilium > Calico (BGP) > Calico (IPIP) > Flannel
# 지연시간 우선: Calico (BGP) > Cilium > Calico (IPIP) > Flannel  
# CPU 사용량: Flannel < Calico < Cilium
# 메모리 사용량: Flannel < Calico < Cilium
```

---

> 💡 **실전 경험**: CNI 선택은 되돌리기 어려운 결정입니다. 초기에 요구사항을 명확히 하고, 테스트 환경에서 충분히 검증한 후 결정하세요. 온프렘에서는 Calico의 BGP 모드가 기존 네트워크와 통합하기 가장 좋습니다.

태그: #cni #networking #calico #flannel #cilium #network-policy