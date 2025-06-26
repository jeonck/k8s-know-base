# Service와 Ingress

## 🎯 개요

쿠버네티스에서 애플리케이션을 외부에 노출하는 핵심 메커니즘인 Service와 Ingress의 설계, 구현, 최적화 방법을 다룹니다. 온프렘 환경에서의 실전 구축 경험과 로드밸런싱 전략을 포함합니다.

## 🔗 Service 유형과 설계

### Service 유형별 특징

| Service 유형 | 사용 사례 | 장점 | 단점 | 온프렘 적합성 |
|--------------|-----------|------|------|---------------|
| **ClusterIP** | 내부 통신 | 기본, 안전 | 외부 접근 불가 | ⭐⭐⭐⭐⭐ |
| **NodePort** | 개발/테스트 | 간단한 외부 노출 | 포트 제한, 보안 취약 | ⭐⭐⭐ |
| **LoadBalancer** | 운영 환경 | 자동 LB 할당 | 클라우드 의존적 | ⭐⭐ |
| **ExternalName** | 외부 서비스 연결 | DNS 기반 | K8s 내부에서만 | ⭐⭐⭐⭐ |

### 기본 Service 구성

```yaml
# basic-services.yaml
---
# ClusterIP Service (기본)
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  namespace: production
  labels:
    app: web-app
    tier: frontend
spec:
  type: ClusterIP
  selector:
    app: web-app
    tier: frontend
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP

---
# NodePort Service (개발 환경용)
apiVersion: v1
kind: Service
metadata:
  name: web-app-nodeport
  namespace: development
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
    nodePort: 30080
    protocol: TCP

---
# Headless Service (StatefulSet용)
apiVersion: v1
kind: Service
metadata:
  name: database-headless
  namespace: production
spec:
  type: ClusterIP
  clusterIP: None  # Headless
  selector:
    app: mysql
    role: primary
  ports:
  - name: mysql
    port: 3306
    targetPort: 3306

---
# ExternalName Service (외부 DB 연결)
apiVersion: v1
kind: Service
metadata:
  name: external-database
  namespace: production
spec:
  type: ExternalName
  externalName: db.company.internal
  ports:
  - name: mysql
    port: 3306
```

### 고급 Service 설정

```yaml
# advanced-services.yaml
---
# Session Affinity가 있는 Service
apiVersion: v1
kind: Service
metadata:
  name: stateful-web-service
  namespace: production
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"  # AWS 예시
spec:
  type: LoadBalancer
  selector:
    app: stateful-web-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
  sessionAffinity: ClientIP  # 세션 어피니티
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600

---
# 다중 포트 Service
apiVersion: v1
kind: Service
metadata:
  name: multi-port-service
  namespace: production
spec:
  selector:
    app: multi-service-app
  ports:
  - name: web
    port: 80
    targetPort: web-port
  - name: api
    port: 8080
    targetPort: api-port
  - name: metrics
    port: 9090
    targetPort: metrics-port

---
# 외부 IP 지정 Service (온프렘용)
apiVersion: v1
kind: Service
metadata:
  name: external-ip-service
  namespace: production
spec:
  selector:
    app: web-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
  externalIPs:
  - 192.168.1.100
  - 192.168.1.101

---
# 로드밸런서 소스 IP 보존
apiVersion: v1
kind: Service
metadata:
  name: source-ip-service
  namespace: production
  annotations:
    service.beta.kubernetes.io/external-traffic: OnlyLocal
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
  externalTrafficPolicy: Local  # 소스 IP 보존
```

## 🌐 온프렘 LoadBalancer 구현

### MetalLB 설치 및 설정

```bash
#!/bin/bash
# install-metallb.sh

install_metallb() {
    echo "Installing MetalLB..."
    
    # MetalLB 네임스페이스 생성
    kubectl create namespace metallb-system
    
    # MetalLB 설치
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
    
    # 설치 완료 대기
    kubectl wait --namespace metallb-system \
        --for=condition=ready pod \
        --selector=app=metallb \
        --timeout=90s
    
    echo "MetalLB installed successfully"
}

configure_metallb_l2() {
    echo "Configuring MetalLB L2 mode..."
    
    cat << 'EOF' | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.250  # 사용 가능한 IP 범위
  autoAssign: true

---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: development-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.2.100-192.168.2.150
  autoAssign: false  # 수동 할당

---
# L2 모드 설정
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - production-pool
  - development-pool
  nodeSelectors:
  - matchLabels:
      metallb-node: "true"  # 특정 노드에서만 광고
EOF
    
    echo "MetalLB L2 configuration applied"
}

configure_metallb_bgp() {
    echo "Configuring MetalLB BGP mode..."
    
    cat << 'EOF' | kubectl apply -f -
apiVersion: metallb.io/v1beta2
kind: BGPPeer
metadata:
  name: router-peer
  namespace: metallb-system
spec:
  myASN: 64512
  peerASN: 64512
  peerAddress: 192.168.1.1  # 라우터 IP
  sourceAddress: 192.168.1.10  # 로컬 IP

---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: bgp-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - production-pool
  communities:
  - 64512:1001
  localPref: 100
EOF
    
    echo "MetalLB BGP configuration applied"
}

install_metallb
configure_metallb_l2
# configure_metallb_bgp  # BGP 사용 시 주석 해제
```

### HAProxy 기반 LoadBalancer

```bash
#!/bin/bash
# setup-haproxy-lb.sh

install_haproxy_lb() {
    echo "Setting up HAProxy LoadBalancer..."
    
    # HAProxy 설정 생성
    cat > /etc/haproxy/haproxy.cfg << 'EOF'
global
    daemon
    maxconn 4096
    log stdout local0 info
    
defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms
    option httplog
    option dontlognull
    option redispatch
    retries 3

# Kubernetes API Server
frontend k8s-api
    bind *:6443
    mode tcp
    default_backend k8s-api-backend

backend k8s-api-backend
    mode tcp
    balance roundrobin
    server master1 192.168.1.10:6443 check
    server master2 192.168.1.11:6443 check
    server master3 192.168.1.12:6443 check

# HTTP/HTTPS 트래픽
frontend web-frontend
    bind *:80
    bind *:443
    mode http
    
    # HTTP to HTTPS 리다이렉트
    redirect scheme https if !{ ssl_fc }
    
    # 애플리케이션별 라우팅
    acl is_web_app hdr(host) -i web.company.com
    acl is_api_app hdr(host) -i api.company.com
    
    use_backend web-app-backend if is_web_app
    use_backend api-app-backend if is_api_app
    default_backend default-backend

backend web-app-backend
    mode http
    balance roundrobin
    option httpchk GET /health
    server worker1 192.168.1.20:30080 check
    server worker2 192.168.1.21:30080 check
    server worker3 192.168.1.22:30080 check

backend api-app-backend
    mode http
    balance roundrobin
    option httpchk GET /api/health
    server worker1 192.168.1.20:30081 check
    server worker2 192.168.1.21:30081 check
    server worker3 192.168.1.22:30081 check

backend default-backend
    mode http
    server default 192.168.1.20:30000 check

# 통계 페이지
frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
    stats admin if TRUE
EOF
    
    # HAProxy 시작
    systemctl enable haproxy
    systemctl start haproxy
    
    echo "HAProxy LoadBalancer configured"
}

install_haproxy_lb
```

## 🚪 Ingress 컨트롤러 구성

### NGINX Ingress Controller 설치

```bash
#!/bin/bash
# install-nginx-ingress.sh

install_nginx_ingress() {
    echo "Installing NGINX Ingress Controller..."
    
    # Ingress-nginx 네임스페이스 생성
    kubectl create namespace ingress-nginx
    
    # NGINX Ingress Controller 설치 (온프렘용)
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/baremetal/deploy.yaml
    
    # 설치 완료 대기
    kubectl wait --namespace ingress-nginx \
        --for=condition=ready pod \
        --selector=app.kubernetes.io/component=controller \
        --timeout=120s
    
    echo "NGINX Ingress Controller installed"
}

configure_nginx_ingress() {
    echo "Configuring NGINX Ingress..."
    
    # ConfigMap으로 NGINX 설정 커스터마이징
    cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
data:
  # 성능 최적화
  worker-processes: "auto"
  worker-connections: "16384"
  max-worker-open-files: "65536"
  
  # 연결 최적화
  keep-alive: "75"
  keep-alive-requests: "1000"
  upstream-keepalive-connections: "50"
  upstream-keepalive-requests: "1000"
  
  # 버퍼 크기 최적화
  client-body-buffer-size: "16k"
  client-header-buffer-size: "1k"
  large-client-header-buffers: "4 16k"
  proxy-buffer-size: "16k"
  proxy-buffers: "8 16k"
  
  # 타임아웃 설정
  client-body-timeout: "60"
  client-header-timeout: "60"
  proxy-connect-timeout: "10"
  proxy-send-timeout: "60"
  proxy-read-timeout: "60"
  
  # 로그 설정
  log-format-upstream: '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" $request_length $request_time [$proxy_upstream_name] [$proxy_alternative_upstream_name] $upstream_addr $upstream_response_length $upstream_response_time $upstream_status $req_id'
  
  # SSL 설정
  ssl-protocols: "TLSv1.2 TLSv1.3"
  ssl-ciphers: "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384"
  ssl-prefer-server-ciphers: "false"
  
  # 보안 헤더
  add-headers: "ingress-nginx/custom-headers"
  
  # 기타 설정
  enable-real-ip: "true"
  forwarded-for-header: "X-Forwarded-For"
  compute-full-forwarded-for: "true"

---
# 커스텀 헤더 ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-headers
  namespace: ingress-nginx
data:
  X-Frame-Options: "SAMEORIGIN"
  X-Content-Type-Options: "nosniff"
  X-XSS-Protection: "1; mode=block"
  Referrer-Policy: "strict-origin-when-cross-origin"
  Content-Security-Policy: "default-src 'self'"
EOF
    
    echo "NGINX Ingress configured"
}

# DaemonSet으로 변경 (모든 노드에 배포)
patch_ingress_daemonset() {
    echo "Patching Ingress to DaemonSet..."
    
    kubectl patch deployment ingress-nginx-controller -n ingress-nginx -p '{"spec":{"template":{"spec":{"hostNetwork":true}}}}'
    
    # DaemonSet으로 변경
    kubectl get deployment ingress-nginx-controller -n ingress-nginx -o yaml | \
    sed 's/kind: Deployment/kind: DaemonSet/' | \
    sed '/replicas:/d' | \
    sed '/strategy:/,+3d' | \
    kubectl apply -f -
    
    kubectl delete deployment ingress-nginx-controller -n ingress-nginx
    
    echo "Ingress Controller converted to DaemonSet"
}

install_nginx_ingress
configure_nginx_ingress
# patch_ingress_daemonset  # 필요 시 주석 해제
```

### Ingress 리소스 설정

```yaml
# ingress-resources.yaml
---
# 기본 Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - web.company.com
    secretName: web-app-tls
  rules:
  - host: web.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-service
            port:
              number: 80

---
# 경로 기반 라우팅 Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.company.com
    secretName: api-tls
  rules:
  - host: api.company.com
    http:
      paths:
      - path: /v1/users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 8080
      - path: /v1/orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 8080
      - path: /v1/payments
        pathType: Prefix
        backend:
          service:
            name: payment-service
            port:
              number: 8080

---
# 인증이 있는 Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: admin-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: admin-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
    nginx.ingress.kubernetes.io/whitelist-source-range: "192.168.1.0/24,10.0.0.0/8"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - admin.company.com
    secretName: admin-tls
  rules:
  - host: admin.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80

---
# 고급 설정 Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advanced-ingress
  namespace: production
  annotations:
    # Rate Limiting
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    
    # 캐싱
    nginx.ingress.kubernetes.io/server-snippet: |
      location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
      }
    
    # CORS 설정
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.company.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "DNT,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization"
    
    # 모니터링
    nginx.ingress.kubernetes.io/enable-modsecurity: "true"
    nginx.ingress.kubernetes.io/enable-owasp-core-rules: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.company.com
    secretName: app-tls
  rules:
  - host: app.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

## 🔐 TLS/SSL 인증서 관리

### cert-manager 설치

```bash
#!/bin/bash
# install-cert-manager.sh

install_cert_manager() {
    echo "Installing cert-manager..."
    
    # cert-manager CRD 설치
    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.crds.yaml
    
    # cert-manager 네임스페이스 생성
    kubectl create namespace cert-manager
    
    # cert-manager 설치
    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml
    
    # 설치 완료 대기
    kubectl wait --namespace cert-manager \
        --for=condition=ready pod \
        --selector=app.kubernetes.io/instance=cert-manager \
        --timeout=120s
    
    echo "cert-manager installed successfully"
}

configure_letsencrypt_issuer() {
    echo "Configuring Let's Encrypt issuer..."
    
    cat << 'EOF' | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: admin@company.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
    - dns01:
        cloudflare:
          email: admin@company.com
          apiTokenSecretRef:
            name: cloudflare-api-token
            key: api-token

---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@company.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
    - dns01:
        cloudflare:
          email: admin@company.com
          apiTokenSecretRef:
            name: cloudflare-api-token
            key: api-token

---
# 자체 서명 인증서 발급자 (개발용)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}

---
# 내부 CA 인증서 발급자
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: ca-issuer
spec:
  ca:
    secretName: ca-key-pair
EOF
    
    echo "Certificate issuers configured"
}

create_wildcard_certificate() {
    echo "Creating wildcard certificate..."
    
    cat << 'EOF' | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-company-com
  namespace: default
spec:
  secretName: wildcard-company-com-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: "*.company.com"
  dnsNames:
  - "*.company.com"
  - "company.com"
EOF
    
    echo "Wildcard certificate requested"
}

install_cert_manager
configure_letsencrypt_issuer
create_wildcard_certificate
```

### 인증서 자동화

```yaml
# certificate-automation.yaml
---
# 자동 인증서 갱신을 위한 Certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: web-app-certificate
  namespace: production
spec:
  secretName: web-app-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: web.company.com
  dnsNames:
  - web.company.com
  - www.web.company.com
  duration: 2160h  # 90일
  renewBefore: 360h  # 15일 전 갱신

---
# Ingress에서 자동 인증서 요청
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auto-tls-ingress
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - auto.company.com
    secretName: auto-tls  # cert-manager가 자동 생성
  rules:
  - host: auto.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: auto-service
            port:
              number: 80
```

## 📊 Service와 Ingress 모니터링

### 모니터링 설정

```yaml
# service-monitoring.yaml
---
# ServiceMonitor (Prometheus Operator)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-ingress-monitor
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
  endpoints:
  - port: prometheus
    interval: 30s
    path: /metrics
    
---
# Service용 Prometheus 메트릭 수집
apiVersion: v1
kind: Service
metadata:
  name: app-service-metrics
  namespace: production
  labels:
    app: web-app
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
spec:
  selector:
    app: web-app
  ports:
  - name: metrics
    port: 9090
    targetPort: 9090
  - name: http
    port: 80
    targetPort: 8080
```

### 모니터링 대시보드

```bash
#!/bin/bash
# service-ingress-monitoring.sh

monitor_service_status() {
    echo "=== Service Status Monitoring ==="
    echo
    
    # 모든 Service 상태
    echo "Service Overview:"
    kubectl get services --all-namespaces -o wide
    echo
    
    # Endpoint 상태 확인
    echo "Endpoint Status:"
    kubectl get endpoints --all-namespaces | while read line; do
        if echo "$line" | grep -q "none"; then
            echo "  ⚠️ $line"
        else
            echo "  ✅ $line"
        fi
    done
    echo
    
    # LoadBalancer 외부 IP 확인
    echo "LoadBalancer External IPs:"
    kubectl get services --all-namespaces -o jsonpath='{range .items[?(@.spec.type=="LoadBalancer")]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.status.loadBalancer.ingress[0].ip}{"\n"}{end}' | \
    while read namespace name ip; do
        if [ -n "$ip" ] && [ "$ip" != "null" ]; then
            echo "  ✅ $namespace/$name: $ip"
        else
            echo "  ⚠️ $namespace/$name: No external IP"
        fi
    done
}

monitor_ingress_status() {
    echo "=== Ingress Status Monitoring ==="
    echo
    
    # Ingress Controller 상태
    echo "Ingress Controller Status:"
    kubectl get pods -n ingress-nginx --no-headers | while read line; do
        pod_name=$(echo "$line" | awk '{print $1}')
        pod_status=$(echo "$line" | awk '{print $3}')
        if [ "$pod_status" = "Running" ]; then
            echo "  ✅ $pod_name: $pod_status"
        else
            echo "  ❌ $pod_name: $pod_status"
        fi
    done
    echo
    
    # Ingress 리소스 상태
    echo "Ingress Resources:"
    kubectl get ingress --all-namespaces -o wide
    echo
    
    # TLS 인증서 상태
    echo "TLS Certificate Status:"
    kubectl get certificates --all-namespaces --no-headers | while read line; do
        namespace=$(echo "$line" | awk '{print $1}')
        name=$(echo "$line" | awk '{print $2}')
        ready=$(echo "$line" | awk '{print $3}')
        if [ "$ready" = "True" ]; then
            echo "  ✅ $namespace/$name: Ready"
        else
            echo "  ⚠️ $namespace/$name: Not Ready"
        fi
    done
}

test_service_connectivity() {
    echo "=== Service Connectivity Test ==="
    echo
    
    # 내부 DNS 해결 테스트
    echo "Internal DNS Resolution:"
    test_services=("kubernetes.default.svc.cluster.local" "kube-dns.kube-system.svc.cluster.local")
    
    for service in "${test_services[@]}"; do
        if kubectl run dns-test-$$ --image=busybox --rm -it --restart=Never -- nslookup "$service" >/dev/null 2>&1; then
            echo "  ✅ $service: DNS OK"
        else
            echo "  ❌ $service: DNS Failed"
        fi
    done
    echo
}

check_ingress_performance() {
    echo "=== Ingress Performance Check ==="
    echo
    
    # NGINX Ingress 메트릭 확인
    echo "NGINX Ingress Metrics:"
    ingress_pod=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o name | head -1)
    if [ -n "$ingress_pod" ]; then
        # 연결 수 확인
        connections=$(kubectl exec -n ingress-nginx "$ingress_pod" -- curl -s localhost:10254/metrics | grep nginx_ingress_controller_nginx_process_connections | head -1)
        echo "  Active Connections: $connections"
        
        # 요청 처리율 확인
        requests=$(kubectl exec -n ingress-nginx "$ingress_pod" -- curl -s localhost:10254/metrics | grep nginx_ingress_controller_requests_total | head -3)
        echo "  Request Metrics:"
        echo "$requests" | while read line; do
            echo "    $line"
        done
    else
        echo "  ⚠️ NGINX Ingress controller not found"
    fi
}

case "${1:-all}" in
    service)
        monitor_service_status
        ;;
    ingress)
        monitor_ingress_status
        ;;
    connectivity)
        test_service_connectivity
        ;;
    performance)
        check_ingress_performance
        ;;
    all)
        monitor_service_status
        echo
        monitor_ingress_status
        echo
        test_service_connectivity
        echo
        check_ingress_performance
        ;;
    *)
        echo "Usage: $0 {service|ingress|connectivity|performance|all}"
        exit 1
        ;;
esac
```

## 🛠️ 고급 라우팅 전략

### 트래픽 분할 (Canary Deployment)

```yaml
# canary-deployment.yaml
---
# 메인 서비스 (90% 트래픽)
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
            name: app-main-service
            port:
              number: 80

---
# Canary 서비스 (10% 트래픽)
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
            name: app-canary-service
            port:
              number: 80
```

### 지역별 라우팅

```yaml
# geo-routing.yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: geo-routing-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/server-snippet: |
      set $region "default";
      if ($geoip_country_code = "KR") {
        set $region "korea";
      }
      if ($geoip_country_code = "JP") {
        set $region "japan";
      }
      if ($geoip_country_code = "US") {
        set $region "usa";
      }
      
      location @korea {
        proxy_pass http://upstream_korea;
      }
      location @japan {
        proxy_pass http://upstream_japan;
      }
      location @usa {
        proxy_pass http://upstream_usa;
      }
spec:
  ingressClassName: nginx
  rules:
  - host: global.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: default-service
            port:
              number: 80
```

## 📋 Best Practices

### Service 설계 원칙

1. **명명 규칙**
   ```
   <application>-<component>-<environment>-service
   예: web-app-frontend-prod-service
   ```

2. **포트 명명**
   ```yaml
   ports:
   - name: http      # HTTP 트래픽
   - name: https     # HTTPS 트래픽  
   - name: grpc      # gRPC 트래픽
   - name: metrics   # 메트릭 수집
   ```

3. **라벨링 전략**
   ```yaml
   labels:
     app.kubernetes.io/name: web-app
     app.kubernetes.io/component: frontend
     app.kubernetes.io/part-of: ecommerce
     app.kubernetes.io/version: "v1.2.3"
   ```

### Ingress 최적화

1. **SSL/TLS 최적화**
   - HTTP/2 활성화
   - OCSP Stapling 구성
   - SSL Session Cache 설정

2. **성능 최적화**
   - Keep-alive 연결 활용
   - 압축 활성화
   - 캐싱 전략 구현

3. **보안 강화**
   - 보안 헤더 설정
   - Rate Limiting 구현
   - WAF 규칙 적용

---

> 💡 **실전 경험**: 온프렘 환경에서 Service와 Ingress 설정의 핵심은 네트워크 토폴로지를 잘 이해하는 것입니다. 특히 LoadBalancer 타입의 Service는 MetalLB 같은 솔루션이 필요하며, Ingress Controller는 DaemonSet으로 배포하여 고가용성을 확보하는 것이 좋습니다. TLS 인증서 관리는 cert-manager를 활용하여 자동화하세요.

태그: #service #ingress #networking #loadbalancer #tls #nginx #metallb #onprem
