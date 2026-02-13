# Module: Kubernetes Deployment Strategies - YAML Implementation Case Study

## 1. 개요 (Overview)

이론적으로 학습한 배포 전략(Rolling, Blue-Green, Canary)은 실제 쿠버네티스(Kubernetes) 매니페스트(YAML) 상에서 구현되는 방식이 확연히 다르다.

본 모듈에서는 각 전략을 구현하기 위해 **어떤 리소스(Resource)의 어떤 필드(Field)**를 수정해야 하는지, 실제 운영 환경에서 사용되는 YAML 코드를 통해 비교 분석한다.

---

## 2. Case 1: 롤링 배포 (Rolling Update) - "Deployment 자체 설정"

롤링 배포는 쿠버네티스 `Deployment` 리소스의 기본 전략이다. 별도의 외부 리소스 없이 **Deployment YAML 내부의 `strategy` 필드**만으로 제어된다.

### 2.1 YAML Manifest 예시

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-api
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 20%        # 1. 추가 생성 허용 범위 (10개 + 2개 = 최대 12개)
      maxUnavailable: 0    # 2. 배포 중 서비스 중단 허용 범위 (0 = 절대 중단 없음)
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: api-server
        image: my-repo/api:v2.0  # 3. 이미지 태그 변경 시 롤링 업데이트 트리거
        readinessProbe:          # 4. 필수: 새 파드가 준비될 때까지 트래픽 차단
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5

```

### 2.2 핵심 분석 포인트

* **제어 위치:** `spec.strategy`
* **동작 방식:** `image` 태그가 변경되면(`v1` -> `v2`), 쿠버네티스 컨트롤러가 `maxSurge`와 `maxUnavailable` 비율에 맞춰 자동으로 파드를 교체한다.
* **실무 팁:** `maxUnavailable: 0`으로 설정해야 진정한 무중단 배포가 보장되지만, 배포 속도는 느려질 수 있다.

---

## 3. Case 2: 블루-그린 배포 (Blue-Green) - "Service Selector 교체"

쿠버네티스에는 `BlueGreen`이라는 전략 타입이 없다. 따라서 **두 개의 Deployment(Blue, Green)**를 동시에 띄워두고, **Service의 `selector`를 수정**하여 트래픽을 순간적으로 전환(Switching)하는 방식을 사용한다.

### 3.1 YAML Manifest 예시 (Service 중심)

**Step 1: Deployment 2개 준비 (Blue & Green)**

```yaml
# 1. Blue (구버전, 현재 운영 중)
kind: Deployment
metadata: {name: app-blue}
spec:
  template:
    metadata: {labels: {app: my-app, version: blue, color: blue}} # Label 분리
---
# 2. Green (신버전, 테스트 대기 중)
kind: Deployment
metadata: {name: app-green}
spec:
  template:
    metadata: {labels: {app: my-app, version: green, color: green}} # Label 분리

```

**Step 2: Service Selector 변경 (트래픽 스위칭)**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: public-service
spec:
  selector:
    app: my-app
    # version: blue   <-- (변경 전) 구버전 바라봄
    version: green    # <-- (변경 후) 이 한 줄을 수정하고 Apply 하면 즉시 전환됨
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080

```

### 3.2 핵심 분석 포인트

* **제어 위치:** `Service`의 `spec.selector`
* **동작 방식:** Deployment 자체를 건드리는 것이 아니라, 서비스가 가리키는 라벨(`version: green`)을 변경함으로써 트래픽의 방향을 튼다.
* **실무 팁:** `kubectl patch service public-service -p '{"spec":{"selector":{"version":"green"}}}'` 명령어로 CI/CD 파이프라인에서 자동화한다.

---

## 4. Case 3: 카나리 배포 (Canary) - "Ingress Annotation 활용"

카나리 배포는 정교한 트래픽 분할(Traffic Splitting)이 필요하다. 가장 일반적인 방식은 **Nginx Ingress Controller**의 Annotation 기능을 활용하는 것이다.

### 4.1 YAML Manifest 예시 (Ingress 중심)

기존 운영용 Ingress(Main) 외에, **Canary용 Ingress를 하나 더 생성**하여 특정 비율만큼 트래픽을 뺏어오는 방식이다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress-canary       # 별도의 카나리용 Ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"        # 1. 카나리 기능 활성화
    nginx.ingress.kubernetes.io/canary-weight: "10"   # 2. 전체 트래픽의 10%만 유입
spec:
  rules:
  - host: my-app.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service-canary  # 3. 신버전(Canary) 서비스로 연결
            port: {number: 80}

```

### 4.2 핵심 분석 포인트

* **제어 위치:** `Ingress`의 `metadata.annotations`
* **동작 방식:** 사용자는 동일한 URL(`my-app.com`)로 접속하지만, 인그레스 컨트롤러가 내부 알고리즘에 의해 10%의 요청을 `app-service-canary`로 라우팅한다.
* **실무 팁:** 단순히 `weight`(비중)만 조절하는 것이 아니라 `canary-by-header`를 사용하여 특정 헤더(예: `Region=Beta`)를 가진 요청만 카나리로 보내는 방식이 테스트에 더 유리하다.

---

## 5. 전략별 YAML 구조 비교 요약

| 비교 항목 | 롤링 (Rolling Update) | 블루-그린 (Blue-Green) | 카나리 (Canary) |
| --- | --- | --- | --- |
| **주 제어 리소스** | **Deployment** | **Service** | **Ingress** (또는 Service Mesh) |
| **변경 트리거** | Image Tag 변경 (`v1`->`v2`) | Selector Label 변경 (`blue`->`green`) | Annotation Weight 변경 (`0`->`10`) |
| **파드 수 변화** | `maxSurge` 만큼 일시 증가 | 배포 시 **2배**로 증가 | 카나리 비율만큼 소폭 증가 |
| **YAML 복잡도** | **하** (기본 설정) | **중** (리소스 2세트 관리) | **상** (Ingress/Annotation 추가) |
| **자동화 도구** | 기본 `kubectl` | ArgoCD, Jenkins | Flagger, Istio |
