# 무중단 배포: Istio를 활용한 Canary Deployment

## ✅ Istio 설치 및 분석 도구 구성

### 🔧 Istio 설치 단계

1. **Istio CLI 설치 및 환경 변수 설정**
    ```bash
    curl -L https://istio.io/downloadIstio | sh -
    cd istio-1.25.1
    export PATH=$PWD/bin:$PATH
    ```

2. **Istio 설치 및 사이드카 주입 설정**
    - **istio-ingressgateway**: 외부 요청을 수신하는 게이트웨이
    - **istio-egressgateway**: 외부 시스템으로 나가는 트래픽을 통제
    - **istiod**: 사이드카 프록시(Envoy) 관리 및 설정 전파

    ```bash
    istioctl install --set profile=demo -y
    kubectl label namespace default istio-injection=enabled
    ```

### 📊 내장 분석 도구 설치 및 대시보드 실행

1. **Kiali (서비스 메쉬 트래픽 시각화)**
    ```bash
    kubectl apply -f samples/addons/kiali.yaml
    istioctl dashboard kiali
    # 또는
    ./bin/istioctl dashboard kiali
    ```

2. **Prometheus (메트릭 수집 및 시각화)**
    ```bash
    kubectl apply -f samples/addons/prometheus.yaml
    istioctl dashboard prometheus
    ```

---

## ✅ Canary 배포 구성

### Step 1: Minikube에 사용할 이미지 빌드

📌 Minikube 내에서 이미지 빌드 및 사용 설정

```bash
eval $(minikube docker-env)       # Minikube Docker 환경 전환
docker build -t my-app:latest .   # 이미지 빌드
eval $(minikube docker-env -u)    # 원래 환경으로 복원
```

- `imagePullPolicy: Never` 로 설정하여 외부 레지스트리에서 이미지를 가져오지 않도록 설정

---

### Step 2: Istio 리소스 구성

#### 🧭 Istio Canary 배포 흐름

1. **Istio Gateway**
    - 외부 트래픽을 수신하는 진입 지점
2. **VirtualService**
    - 트래픽을 내부 서비스(Pod)로 라우팅하고, 버전별로 트래픽 분배

#### 1. Deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v1 
  labels:
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
        - name: myapp-v1
          image: myapp:1.0
          imagePullPolicy: Never
          ports:
            - containerPort: 8001
```

#### 2. Service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  labels:
    app: myapp
spec:
  ports:
    - port: 8001
      targetPort: 8001
  selector:
    app: myapp
```

#### 3. Gateway.yaml
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: hello-world-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

✅ 게이트웨이 확인
```bash
kubectl get gateway
```

#### 4. VirtualService.yaml
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: hello-world-virtual-services
spec:
  hosts:
    - "*"
  gateways:
    - hello-world-gateway
  http:
    - match:
        - uri:
            exact: /
      route:
        - destination:
            host: myapp-service
            subset: v1
          weight: 50
        - destination:
            host: myapp-service
            subset: v2
          weight: 50
```

#### 5. DestinationRule.yaml
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: hello-world-destination-rule
spec:
  host: myapp-service
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

✅ DestinationRule 확인
```bash
kubectl get destinationrule
```

---

## 🧼 Istio 클러스터 정리

```bash
cd /home/ubuntu/app/istio-1.25.1
istioctl uninstall -y --purge
kubectl delete namespace istio-system

helm uninstall canary
minikube stop
```
