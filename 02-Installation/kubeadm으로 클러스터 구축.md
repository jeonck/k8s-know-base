# kubeadm으로 클러스터 구축

## 🎯 개요

kubeadm을 사용하여 온프렘 환경에서 프로덕션 수준의 쿠버네티스 클러스터를 구축하는 상세 가이드입니다. 단일 마스터부터 고가용성 멀티 마스터 구성까지 단계별로 설명합니다.

## 🔧 사전 준비사항

### 하드웨어 요구사항

```bash
# 최소 요구사항
마스터 노드: 2 CPU, 4GB RAM, 50GB Storage
워커 노드: 2 CPU, 4GB RAM, 50GB Storage

# 권장 사양
마스터 노드: 4 CPU, 8GB RAM, 100GB SSD
워커 노드: 8 CPU, 16GB RAM, 200GB SSD
```

### 네트워크 요구사항

```bash
# 필수 포트 오픈
마스터 노드:
- 6443/tcp: Kubernetes API server
- 2379-2380/tcp: etcd server client API
- 10250/tcp: kubelet API
- 10251/tcp: kube-scheduler
- 10252/tcp: kube-controller-manager

워커 노드:
- 10250/tcp: kubelet API
- 30000-32767/tcp: NodePort Services
```

## 🛠️ 시스템 초기 설정

### 모든 노드 공통 설정

```bash
#!/bin/bash
# prepare-nodes.sh - 모든 노드에서 실행

set -e

echo "=== Kubernetes 노드 초기화 시작 ==="

# 1. 시스템 업데이트
update_system() {
    echo "시스템 업데이트 중..."
    
    # CentOS/RHEL
    if command -v yum >/dev/null 2>&1; then
        yum update -y
        yum install -y epel-release
        yum install -y wget curl net-tools bind-utils
    fi
    
    # Ubuntu/Debian
    if command -v apt >/dev/null 2>&1; then
        apt update && apt upgrade -y
        apt install -y apt-transport-https ca-certificates curl gnupg lsb-release wget
    fi
}

# 2. SELinux 및 방화벽 설정
configure_security() {
    echo "보안 설정 구성 중..."
    
    # SELinux 비활성화 (CentOS/RHEL)
    if [ -f /etc/selinux/config ]; then
        setenforce 0 2>/dev/null || true
        sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
    fi
    
    # 방화벽 비활성화 (개발 환경) 또는 포트 오픈 (운영 환경)
    if systemctl is-active --quiet firewalld; then
        # 개발 환경: 방화벽 비활성화
        # systemctl disable firewalld
        # systemctl stop firewalld
        
        # 운영 환경: 필요한 포트만 오픈
        firewall-cmd --permanent --add-port=6443/tcp
        firewall-cmd --permanent --add-port=2379-2380/tcp
        firewall-cmd --permanent --add-port=10250/tcp
        firewall-cmd --permanent --add-port=10251/tcp
        firewall-cmd --permanent --add-port=10252/tcp
        firewall-cmd --permanent --add-port=30000-32767/tcp
        firewall-cmd --reload
    fi
    
    # UFW 설정 (Ubuntu)
    if command -v ufw >/dev/null 2>&1; then
        ufw allow 6443/tcp
        ufw allow 2379:2380/tcp
        ufw allow 10250/tcp
        ufw allow 10251/tcp
        ufw allow 10252/tcp
        ufw allow 30000:32767/tcp
    fi
}

# 3. Swap 비활성화
disable_swap() {
    echo "Swap 비활성화 중..."
    
    # 즉시 비활성화
    swapoff -a
    
    # 영구 비활성화
    sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
    
    # 확인
    free -h
}

# 4. 커널 모듈 로드
load_kernel_modules() {
    echo "필수 커널 모듈 로드 중..."
    
    cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
    
    modprobe overlay
    modprobe br_netfilter
}

# 5. sysctl 파라미터 설정
configure_sysctl() {
    echo "네트워크 파라미터 설정 중..."
    
    cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
    
    sysctl --system
}

# 6. 컨테이너 런타임 설치 (containerd)
install_containerd() {
    echo "containerd 설치 중..."
    
    # Docker 저장소 추가
    if command -v yum >/dev/null 2>&1; then
        yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
        yum install -y containerd.io
    fi
    
    if command -v apt >/dev/null 2>&1; then
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
        echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
        apt update
        apt install -y containerd.io
    fi
    
    # containerd 설정
    mkdir -p /etc/containerd
    containerd config default | tee /etc/containerd/config.toml
    
    # systemd cgroup 드라이버 설정
    sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
    
    # containerd 시작
    systemctl enable containerd
    systemctl start containerd
    systemctl status containerd
}

# 7. kubeadm, kubelet, kubectl 설치
install_kubernetes_tools() {
    echo "Kubernetes 도구 설치 중..."
    
    # 저장소 추가
    if command -v yum >/dev/null 2>&1; then
        cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
        
        yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
    fi
    
    if command -v apt >/dev/null 2>&1; then
        curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
        cat <<EOF | tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
        
        apt update
        apt install -y kubelet kubeadm kubectl
        apt-mark hold kubelet kubeadm kubectl
    fi
    
    # kubelet 활성화
    systemctl enable kubelet
}

# 8. 노드 정보 설정
configure_node_info() {
    echo "노드 정보 설정 중..."
    
    # 호스트명 설정 (필요한 경우)
    read -p "호스트명을 변경하시겠습니까? (y/N): " -n 1 -r
    echo
    if [[ $REPLY =~ ^[Yy]$ ]]; then
        read -p "새 호스트명 입력: " new_hostname
        hostnamectl set-hostname "$new_hostname"
        echo "127.0.0.1 $new_hostname" >> /etc/hosts
    fi
    
    # IP 주소 확인
    local_ip=$(ip route get 8.8.8.8 | head -1 | awk '{print $7}')
    echo "현재 노드 IP: $local_ip"
    
    # /etc/hosts 파일 업데이트 권장사항 표시
    echo ""
    echo "=== 권장사항 ==="
    echo "모든 노드의 /etc/hosts 파일에 다음과 같이 추가하세요:"
    echo "# Kubernetes Cluster Nodes"
    echo "192.168.1.10 k8s-master-01"
    echo "192.168.1.11 k8s-master-02"
    echo "192.168.1.12 k8s-master-03"
    echo "192.168.1.20 k8s-worker-01"
    echo "192.168.1.21 k8s-worker-02"
    echo "192.168.1.22 k8s-worker-03"
}

# 실행
main() {
    update_system
    configure_security
    disable_swap
    load_kernel_modules
    configure_sysctl
    install_containerd
    install_kubernetes_tools
    configure_node_info
    
    echo ""
    echo "=== 노드 초기화 완료 ==="
    echo "다음 단계: 마스터 노드에서 'kubeadm init' 실행"
    echo "설치된 버전:"
    kubeadm version
    kubelet --version
    kubectl version --client
}

main "$@"
```

## 🏆 단일 마스터 클러스터 구축

### 마스터 노드 초기화

```bash
#!/bin/bash
# init-master.sh

set -e

KUBERNETES_VERSION="1.28.0"
POD_CIDR="10.244.0.0/16"
SERVICE_CIDR="10.96.0.0/12"
API_SERVER_ADVERTISE_ADDRESS=""

# 설정값 입력
configure_cluster() {
    echo "=== 클러스터 설정 ==="
    
    # API 서버 주소 설정
    local_ip=$(ip route get 8.8.8.8 | head -1 | awk '{print $7}')
    read -p "API 서버 주소 입력 (기본값: $local_ip): " api_address
    API_SERVER_ADVERTISE_ADDRESS=${api_address:-$local_ip}
    
    # Pod CIDR 확인
    read -p "Pod CIDR (기본값: $POD_CIDR): " pod_cidr
    POD_CIDR=${pod_cidr:-$POD_CIDR}
    
    # Service CIDR 확인
    read -p "Service CIDR (기본값: $SERVICE_CIDR): " service_cidr
    SERVICE_CIDR=${service_cidr:-$SERVICE_CIDR}
    
    echo "설정 확인:"
    echo "  API Server: $API_SERVER_ADVERTISE_ADDRESS"
    echo "  Pod CIDR: $POD_CIDR"
    echo "  Service CIDR: $SERVICE_CIDR"
    echo "  Kubernetes Version: $KUBERNETES_VERSION"
}

# kubeadm 설정 파일 생성
create_kubeadm_config() {
    echo "kubeadm 설정 파일 생성 중..."
    
    cat > kubeadm-config.yaml << EOF
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: $API_SERVER_ADVERTISE_ADDRESS
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  kubeletExtraArgs:
    node-ip: $API_SERVER_ADVERTISE_ADDRESS

---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v$KUBERNETES_VERSION
clusterName: kubernetes
controlPlaneEndpoint: $API_SERVER_ADVERTISE_ADDRESS:6443
networking:
  serviceSubnet: $SERVICE_CIDR
  podSubnet: $POD_CIDR
  dnsDomain: cluster.local

# API Server 설정
apiServer:
  extraArgs:
    audit-log-maxage: "30"
    audit-log-maxbackup: "3"
    audit-log-maxsize: "100"
    audit-log-path: /var/log/audit.log
    enable-admission-plugins: NodeRestriction
  extraVolumes:
  - name: audit-log
    hostPath: /var/log/audit.log
    mountPath: /var/log/audit.log
    pathType: FileOrCreate

# Controller Manager 설정
controllerManager:
  extraArgs:
    bind-address: 0.0.0.0

# Scheduler 설정
scheduler:
  extraArgs:
    bind-address: 0.0.0.0

# etcd 설정
etcd:
  local:
    extraArgs:
      listen-metrics-urls: http://0.0.0.0:2381

---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
failSwapOn: false
authentication:
  webhook:
    enabled: true
authorization:
  mode: Webhook
clusterDomain: cluster.local
clusterDNS:
- 10.96.0.10
runtimeRequestTimeout: 2m
rotateCertificates: true
serverTLSBootstrap: true
staticPodPath: /etc/kubernetes/manifests
EOF
    
    echo "kubeadm 설정 파일이 생성되었습니다: kubeadm-config.yaml"
}

# 클러스터 초기화
initialize_cluster() {
    echo "=== 클러스터 초기화 시작 ==="
    
    # kubeadm init 실행
    kubeadm init --config=kubeadm-config.yaml --upload-certs
    
    if [ $? -eq 0 ]; then
        echo "✅ 클러스터 초기화 성공"
    else
        echo "❌ 클러스터 초기화 실패"
        exit 1
    fi
}

# kubectl 설정
setup_kubectl() {
    echo "kubectl 설정 중..."
    
    # root 사용자용
    export KUBECONFIG=/etc/kubernetes/admin.conf
    echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> ~/.bashrc
    
    # 일반 사용자용 (옵션)
    read -p "일반 사용자용 kubectl을 설정하시겠습니까? (y/N): " -n 1 -r
    echo
    if [[ $REPLY =~ ^[Yy]$ ]]; then
        read -p "사용자명 입력: " username
        if id "$username" &>/dev/null; then
            mkdir -p /home/$username/.kube
            cp -i /etc/kubernetes/admin.conf /home/$username/.kube/config
            chown $(id -u $username):$(id -g $username) /home/$username/.kube/config
            echo "✅ $username 사용자용 kubectl 설정 완료"
        else
            echo "❌ 사용자 $username을 찾을 수 없습니다"
        fi
    fi
}

# CNI 플러그인 설치 (Calico)
install_cni() {
    echo "=== CNI 플러그인 설치 (Calico) ==="
    
    # Calico operator 설치
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
    
    # Calico 설정
    cat > calico-config.yaml << EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: $POD_CIDR
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()

---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF
    
    kubectl apply -f calico-config.yaml
    
    echo "Calico 설치 대기 중..."
    kubectl wait --for=condition=Ready pods --all -n calico-system --timeout=300s
    
    echo "✅ CNI 플러그인 설치 완료"
}

# 마스터 노드 스케줄링 허용 (단일 노드 클러스터인 경우)
allow_master_scheduling() {
    read -p "마스터 노드에서도 Pod 스케줄링을 허용하시겠습니까? (단일 노드 테스트용) (y/N): " -n 1 -r
    echo
    if [[ $REPLY =~ ^[Yy]$ ]]; then
        kubectl taint nodes --all node-role.kubernetes.io/control-plane-
        echo "✅ 마스터 노드 스케줄링 허용됨"
    fi
}

# 클러스터 상태 확인
verify_cluster() {
    echo "=== 클러스터 상태 확인 ==="
    
    echo "노드 상태:"
    kubectl get nodes -o wide
    
    echo ""
    echo "시스템 Pod 상태:"
    kubectl get pods -n kube-system
    
    echo ""
    echo "Calico Pod 상태:"
    kubectl get pods -n calico-system
    
    echo ""
    echo "클러스터 정보:"
    kubectl cluster-info
}

# 워커 노드 조인 명령어 생성
generate_join_command() {
    echo "=== 워커 노드 조인 명령어 ==="
    
    join_command=$(kubeadm token create --print-join-command)
    
    echo "다음 명령어를 워커 노드에서 실행하세요:"
    echo ""
    echo "$join_command"
    echo ""
    
    # 파일로도 저장
    echo "$join_command" > join-command.sh
    chmod +x join-command.sh
    echo "조인 명령어가 join-command.sh 파일로 저장되었습니다."
}

# 메인 실행
main() {
    configure_cluster
    create_kubeadm_config
    initialize_cluster
    setup_kubectl
    install_cni
    allow_master_scheduling
    verify_cluster
    generate_join_command
    
    echo ""
    echo "=== 🎉 단일 마스터 클러스터 구축 완료 ==="
    echo ""
    echo "다음 단계:"
    echo "1. 워커 노드에서 join-command.sh 실행"
    echo "2. kubectl get nodes로 노드 상태 확인"
    echo "3. 애플리케이션 배포 테스트"
    echo ""
    echo "유용한 명령어:"
    echo "  kubectl get nodes"
    echo "  kubectl get pods --all-namespaces"
    echo "  kubectl cluster-info"
}

main "$@"
```

## 🔗 워커 노드 조인

### 워커 노드 설정

```bash
#!/bin/bash
# join-worker.sh

set -e

JOIN_COMMAND=""
MASTER_IP=""

# 조인 명령어 입력
get_join_command() {
    echo "=== 워커 노드 조인 설정 ==="
    
    if [ -f "join-command.sh" ]; then
        echo "기존 조인 명령어 파일을 찾았습니다."
        read -p "join-command.sh 파일을 사용하시겠습니까? (y/N): " -n 1 -r
        echo
        if [[ $REPLY =~ ^[Yy]$ ]]; then
            JOIN_COMMAND=$(cat join-command.sh)
        fi
    fi
    
    if [ -z "$JOIN_COMMAND" ]; then
        echo "마스터 노드에서 생성된 조인 명령어를 입력하세요:"
        echo "예: kubeadm join 192.168.1.10:6443 --token abc123... --discovery-token-ca-cert-hash sha256:def456..."
        read -p "조인 명령어: " JOIN_COMMAND
    fi
    
    # 마스터 IP 추출
    MASTER_IP=$(echo "$JOIN_COMMAND" | grep -o '[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}')
}

# 마스터 노드 연결 확인
verify_master_connection() {
    echo "마스터 노드 연결 확인 중..."
    
    if ping -c 3 "$MASTER_IP" >/dev/null 2>&1; then
        echo "✅ 마스터 노드 연결 가능: $MASTER_IP"
    else
        echo "❌ 마스터 노드 연결 실패: $MASTER_IP"
        exit 1
    fi
    
    # API 서버 포트 확인
    if timeout 5 bash -c "</dev/tcp/$MASTER_IP/6443" >/dev/null 2>&1; then
        echo "✅ API 서버 포트 접근 가능: $MASTER_IP:6443"
    else
        echo "❌ API 서버 포트 접근 실패: $MASTER_IP:6443"
        echo "방화벽 설정을 확인하세요."
        exit 1
    fi
}

# 노드 정보 설정
configure_node() {
    echo "노드 정보 설정 중..."
    
    # 노드명 설정 (옵션)
    read -p "워커 노드 이름을 설정하시겠습니까? (현재: $(hostname)) (y/N): " -n 1 -r
    echo
    if [[ $REPLY =~ ^[Yy]$ ]]; then
        read -p "새 노드 이름: " node_name
        hostnamectl set-hostname "$node_name"
        echo "✅ 노드 이름 변경: $node_name"
    fi
    
    # kubelet 추가 설정
    local_ip=$(ip route get 8.8.8.8 | head -1 | awk '{print $7}')
    
    mkdir -p /etc/systemd/system/kubelet.service.d
    cat > /etc/systemd/system/kubelet.service.d/20-node-ip.conf << EOF
[Service]
Environment="KUBELET_EXTRA_ARGS=--node-ip=$local_ip"
EOF
    
    systemctl daemon-reload
    echo "✅ kubelet 노드 IP 설정: $local_ip"
}

# 클러스터 조인 실행
join_cluster() {
    echo "=== 클러스터 조인 실행 ==="
    echo "명령어: $JOIN_COMMAND"
    
    # 조인 실행
    eval "$JOIN_COMMAND"
    
    if [ $? -eq 0 ]; then
        echo "✅ 클러스터 조인 성공"
    else
        echo "❌ 클러스터 조인 실패"
        exit 1
    fi
}

# 조인 상태 확인
verify_join() {
    echo "=== 조인 상태 확인 ==="
    
    # kubelet 상태 확인
    if systemctl is-active --quiet kubelet; then
        echo "✅ kubelet 서비스 실행 중"
    else
        echo "❌ kubelet 서비스 실행 실패"
        systemctl status kubelet
        return 1
    fi
    
    # 노드 등록 확인 (마스터에서 확인 필요)
    echo ""
    echo "노드 등록 확인을 위해 마스터 노드에서 다음 명령어를 실행하세요:"
    echo "  kubectl get nodes"
    echo ""
    
    # Pod 네트워킹 테스트
    echo "Pod 네트워킹 테스트 중..."
    timeout 60 bash -c 'while ! ls /opt/cni/bin/ >/dev/null 2>&1; do sleep 5; done'
    
    if [ $? -eq 0 ]; then
        echo "✅ CNI 플러그인 준비됨"
    else
        echo "⚠️ CNI 플러그인 대기 중..."
    fi
}

# 메인 실행
main() {
    get_join_command
    verify_master_connection
    configure_node
    join_cluster
    verify_join
    
    echo ""
    echo "=== 🎉 워커 노드 조인 완료 ==="
    echo ""
    echo "다음 단계:"
    echo "1. 마스터 노드에서 'kubectl get nodes' 실행하여 노드 확인"
    echo "2. 모든 노드가 Ready 상태가 될 때까지 대기"
    echo "3. 애플리케이션 배포 테스트"
    echo ""
    echo "문제 해결:"
    echo "  journalctl -u kubelet -f  # kubelet 로그 확인"
    echo "  systemctl status kubelet  # kubelet 상태 확인"
}

main "$@"
```

## 🏗️ 고가용성 클러스터 구축

### HA 클러스터 설정

```bash
#!/bin/bash
# init-ha-cluster.sh

set -e

# HA 클러스터 설정
CONTROL_PLANE_ENDPOINT="k8s-api.company.local:6443"  # 로드밸런서 주소
KUBERNETES_VERSION="1.28.0"
POD_CIDR="10.244.0.0/16"
SERVICE_CIDR="10.96.0.0/12"

# 첫 번째 마스터 노드 초기화
init_first_master() {
    echo "=== 첫 번째 마스터 노드 초기화 ==="
    
    local_ip=$(ip route get 8.8.8.8 | head -1 | awk '{print $7}')
    
    cat > kubeadm-ha-config.yaml << EOF
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: $local_ip
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock

---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v$KUBERNETES_VERSION
clusterName: kubernetes
controlPlaneEndpoint: $CONTROL_PLANE_ENDPOINT
networking:
  serviceSubnet: $SERVICE_CIDR
  podSubnet: $POD_CIDR
  dnsDomain: cluster.local

# etcd 외부 설정 (옵션 - 별도 etcd 클러스터 사용 시)
# etcd:
#   external:
#     endpoints:
#     - https://etcd1.company.local:2379
#     - https://etcd2.company.local:2379
#     - https://etcd3.company.local:2379
#     caFile: /etc/etcd/ca.crt
#     certFile: /etc/etcd/client.crt
#     keyFile: /etc/etcd/client.key

apiServer:
  extraArgs:
    audit-log-maxage: "30"
    audit-log-maxbackup: "10"
    audit-log-maxsize: "100"
    audit-log-path: /var/log/audit.log
    enable-admission-plugins: NodeRestriction,LimitRanger,ResourceQuota
  certSANs:
  - k8s-api.company.local
  - 192.168.1.5  # 로드밸런서 IP
  - 192.168.1.10 # 마스터 1 IP
  - 192.168.1.11 # 마스터 2 IP
  - 192.168.1.12 # 마스터 3 IP

controllerManager:
  extraArgs:
    bind-address: 0.0.0.0

scheduler:
  extraArgs:
    bind-address: 0.0.0.0

---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
failSwapOn: false
clusterDomain: cluster.local
clusterDNS:
- 10.96.0.10
EOF
    
    # 첫 번째 마스터 초기화
    kubeadm init --config=kubeadm-ha-config.yaml --upload-certs
    
    # kubectl 설정
    export KUBECONFIG=/etc/kubernetes/admin.conf
    echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> ~/.bashrc
    
    # CNI 설치
    kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
    
    cat > calico-ha-config.yaml << EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: $POD_CIDR
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
EOF
    
    kubectl apply -f calico-ha-config.yaml
    
    echo "✅ 첫 번째 마스터 노드 초기화 완료"
}

# 추가 마스터 조인 명령어 생성
generate_master_join_command() {
    echo "=== 추가 마스터 조인 명령어 생성 ==="
    
    # 인증서 키 생성
    certificate_key=$(kubeadm init phase upload-certs --upload-certs | tail -1)
    
    # 토큰 생성
    token=$(kubeadm token create)
    
    # CA 인증서 해시 생성
    ca_cert_hash=$(openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //')
    
    # 조인 명령어 생성
    master_join_command="kubeadm join $CONTROL_PLANE_ENDPOINT --token $token --discovery-token-ca-cert-hash sha256:$ca_cert_hash --control-plane --certificate-key $certificate_key"
    
    echo "추가 마스터 노드에서 실행할 명령어:"
    echo "$master_join_command"
    echo
    
    # 파일로 저장
    echo "$master_join_command" > master-join-command.sh
    chmod +x master-join-command.sh
    
    # 워커 노드 조인 명령어도 생성
    worker_join_command="kubeadm join $CONTROL_PLANE_ENDPOINT --token $token --discovery-token-ca-cert-hash sha256:$ca_cert_hash"
    echo "$worker_join_command" > worker-join-command.sh
    chmod +x worker-join-command.sh
    
    echo "✅ 조인 명령어 파일 생성 완료"
    echo "  - master-join-command.sh: 추가 마스터 노드용"
    echo "  - worker-join-command.sh: 워커 노드용"
}

# 로드밸런서 설정 가이드
show_loadbalancer_guide() {
    echo "=== 로드밸런서 설정 가이드 ==="
    echo ""
    echo "HA 클러스터를 위해 로드밸런서 설정이 필요합니다."
    echo ""
    echo "HAProxy 설정 예시:"
    cat << 'EOF'
# /etc/haproxy/haproxy.cfg
global
    daemon
    maxconn 4096

defaults
    mode tcp
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend k8s-api
    bind *:6443
    default_backend k8s-api-backend

backend k8s-api-backend
    balance roundrobin
    server master1 192.168.1.10:6443 check
    server master2 192.168.1.11:6443 check
    server master3 192.168.1.12:6443 check
EOF
    echo ""
    echo "NGINX 설정 예시:"
    cat << 'EOF'
# /etc/nginx/nginx.conf
stream {
    upstream k8s-api {
        server 192.168.1.10:6443;
        server 192.168.1.11:6443;
        server 192.168.1.12:6443;
    }

    server {
        listen 6443;
        proxy_pass k8s-api;
        proxy_timeout 3s;
        proxy_responses 1;
    }
}
EOF
    echo ""
}

# HA 클러스터 상태 확인
verify_ha_cluster() {
    echo "=== HA 클러스터 상태 확인 ==="
    
    # 노드 상태
    echo "노드 상태:"
    kubectl get nodes -o wide
    echo
    
    # etcd 상태
    echo "etcd 클러스터 상태:"
    kubectl exec -n kube-system etcd-$(hostname) -- etcdctl \
        --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/server.crt \
        --key=/etc/kubernetes/pki/etcd/server.key \
        endpoint status --write-out=table
    echo
    
    # API 서버 상태
    echo "API 서버 상태:"
    kubectl get endpoints kubernetes -o wide
    echo
    
    # 시스템 Pod 상태
    echo "시스템 Pod 상태:"
    kubectl get pods -n kube-system -o wide | grep -E "(api|controller|scheduler|etcd)"
}

# 메인 실행
main() {
    echo "=== HA 쿠버네티스 클러스터 구축 ==="
    echo ""
    
    show_loadbalancer_guide
    read -p "로드밸런서 설정이 완료되었습니까? (y/N): " -n 1 -r
    echo
    if [[ ! $REPLY =~ ^[Yy]$ ]]; then
        echo "먼저 로드밸런서를 설정한 후 다시 실행하세요."
        exit 1
    fi
    
    init_first_master
    generate_master_join_command
    verify_ha_cluster
    
    echo ""
    echo "=== 🎉 HA 클러스터 첫 번째 마스터 구축 완료 ==="
    echo ""
    echo "다음 단계:"
    echo "1. 추가 마스터 노드에서 master-join-command.sh 실행"
    echo "2. 워커 노드에서 worker-join-command.sh 실행"
    echo "3. 모든 노드가 Ready 상태 확인"
    echo "4. 로드밸런서 헬스체크 확인"
}

main "$@"
```

## 🔧 클러스터 검증과 테스트

### 클러스터 검증 스크립트

```bash
#!/bin/bash
# verify-cluster.sh

echo "=== 쿠버네티스 클러스터 검증 ==="

# 1. 기본 상태 확인
basic_checks() {
    echo "1. 기본 상태 확인"
    
    echo "  노드 상태:"
    kubectl get nodes -o wide
    
    echo "  시스템 Pod 상태:"
    kubectl get pods -n kube-system
    
    echo "  클러스터 정보:"
    kubectl cluster-info
}

# 2. 네트워킹 테스트
test_networking() {
    echo "2. 네트워킹 테스트"
    
    # DNS 테스트
    echo "  DNS 테스트:"
    kubectl run dns-test --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default
    
    # Pod 간 통신 테스트
    echo "  Pod 간 통신 테스트:"
    kubectl run netshoot-1 --image=nicolaka/netshoot --rm -it --restart=Never -- ping -c 3 8.8.8.8
}

# 3. 스토리지 테스트
test_storage() {
    echo "3. 스토리지 테스트"
    
    cat << EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
    
    kubectl get pvc test-pvc
    kubectl delete pvc test-pvc
}

# 4. 애플리케이션 배포 테스트
test_application() {
    echo "4. 애플리케이션 배포 테스트"
    
    # 테스트 deployment 생성
    kubectl create deployment nginx-test --image=nginx
    kubectl scale deployment nginx-test --replicas=3
    kubectl expose deployment nginx-test --port=80 --target-port=80
    
    # 상태 확인
    kubectl rollout status deployment/nginx-test
    kubectl get pods -l app=nginx-test
    kubectl get service nginx-test
    
    # 정리
    kubectl delete deployment nginx-test
    kubectl delete service nginx-test
}

# 실행
basic_checks
echo
test_networking
echo
test_storage
echo
test_application

echo ""
echo "✅ 클러스터 검증 완료"
```

## 📋 문제 해결 가이드

### 일반적인 문제와 해결책

```bash
# 1. kubelet이 시작되지 않는 경우
systemctl status kubelet
journalctl -u kubelet -f

# 해결책:
# - swap 비활성화 확인: swapoff -a
# - containerd 상태 확인: systemctl status containerd
# - 포트 충돌 확인: netstat -tulpn | grep :10250

# 2. Pod가 Pending 상태인 경우
kubectl describe pod <pod-name>

# 해결책:
# - 노드 리소스 확인: kubectl describe nodes
# - CNI 플러그인 상태 확인: kubectl get pods -n kube-system
# - Taint 확인: kubectl describe node <node-name>

# 3. DNS 해결이 안 되는 경우
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 해결책:
# - CoreDNS Pod 재시작: kubectl delete pod -n kube-system -l k8s-app=kube-dns
# - DNS 설정 확인: kubectl get configmap coredns -n kube-system -o yaml

# 4. 이미지 풀이 안 되는 경우
kubectl describe pod <pod-name>

# 해결책:
# - 이미지 레지스트리 접근 확인
# - imagePullSecrets 설정 확인
# - containerd 설정 확인: /etc/containerd/config.toml
```

---

> 💡 **실전 경험**: kubeadm을 사용한 클러스터 구축에서 가장 중요한 것은 사전 준비입니다. 특히 네트워크 설정, 방화벽 규칙, 그리고 시간 동기화를 확실히 해야 합니다. HA 클러스터를 구축할 때는 로드밸런서 설정이 매우 중요하며, 인증서 SAN에 모든 필요한 IP와 도메인을 포함해야 합니다.

태그: #kubeadm #cluster #installation #ha #onprem #setup
