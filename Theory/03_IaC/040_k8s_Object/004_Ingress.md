## 인그레스(Ingress)의 상세 분석

**인그레스(Ingress)**는 클러스터 외부에서 내부 서비스로 접근하는 HTTP와 HTTPS 요청을 관리하는 API 오브젝트입니다. 서비스(Service)가 L4 계층(IP, Port) 수준의 전달자라면, 인그레스는 **L7 계층(애플리케이션 계층)** 수준에서 도메인 기반 가상 호스팅, URL 경로 기반 라우팅, SSL/TLS 보안 인증 등을 처리하는 '스마트한 입구' 역할을 합니다.

---

### 1. 인그레스가 필요한 이유 (비용 및 관리 효율)

강사님은 수강생들에게 인그레스의 도입 이유를 **비용**과 **관리** 측면에서 설명하시면 효과적입니다.

* **비용 절감**: 서비스마다 `type: LoadBalancer`를 생성하면 개별적인 AWS ELB 비용이 발생하지만, 인그레스를 사용하면 단 하나의 로드밸런서로 여러 서비스를 통합 관리할 수 있습니다.
* **라우팅 규칙 통합**: URL 경로(` /api`, `/static`)나 도메인(`shop.com`, `admin.com`)에 따라 트래픽을 분산하는 규칙을 하나의 설정 파일에서 선언적으로 관리합니다.
* **보안 인증 통합**: 각 서비스마다 SSL 설정을 할 필요 없이 인그레스 계층에서 인증서를 일괄 적용(TLS Termination)할 수 있습니다.

---

### 2. 인그레스의 구성 요소

인그레스가 작동하려면 두 가지가 모두 필요합니다.

1. **인그레스 리소스 (Ingress Resource)**: 트래픽 라우팅 규칙을 정의한 YAML 파일입니다.
2. **인그레스 컨트롤러 (Ingress Controller)**: 리소스에 정의된 규칙을 실제로 실행하는 엔진입니다. (예: Nginx Ingress Controller, AWS Load Balancer Controller)
* *참고: 쿠버네티스 클러스터는 기본적으로 컨트롤러를 포함하지 않으므로 별도로 설치해야 합니다.*



---

### 3. 인그레스 YAML 구조 (경로 기반 라우팅 예시)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    # 컨트롤러 종류 지정 (예: nginx)
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: my-it-class.com             # 도메인 설정
    http:
      paths:
      - path: /order                 # URL 경로가 /order인 경우
        pathType: Prefix
        backend:
          service:
            name: order-service      # 주문 서비스로 전달
            port:
              number: 80
      - path: /billing               # URL 경로가 /billing인 경우
        pathType: Prefix
        backend:
          service:
            name: billing-service    # 결제 서비스로 전달
            port:
              number: 80

```

---

### 4. 주요 라우팅 방식

| 방식 | 설명 | 예시 |
| --- | --- | --- |
| **Host 기반** | 요청된 도메인 이름에 따라 다른 서비스로 연결 | `a.com` -> Service A, `b.com` -> Service B |
| **Path 기반** | 동일 도메인 내의 URL 경로에 따라 연결 | `test.com/v1` -> Service V1, `test.com/v2` -> Service V2 |
| **TLS/SSL** | 443 포트 통신 및 인증서 관리 | HTTPS 통신 적용 및 인증서 종단 처리 |

---

### 💡 실전 팁: 컨트롤러 선택 가이드

실무 환경이 AWS라면 **AWS Load Balancer Controller**를 사용하여 인그레스 생성 시 자동으로 AWS의 **ALB(Application Load Balancer)**가 생성되도록 구성하는 것이 정석입니다. 반면 온프레미스나 범용적인 환경에서는 **Nginx Ingress Controller**가 가장 많이 쓰이며 세밀한 설정(Annotation)이 가능하다는 점을 강조해 주세요.

---

### ⚠️ 비용 및 유료 전환 주의사항

* **ALB 과금**: AWS Load Balancer Controller를 통해 인그레스를 생성하면 AWS **ALB 비용**이 발생합니다. ALB는 인스턴스 시간당 요금 외에도 LcU(Load Balancer Capacity Units)라는 복잡한 과금 체계를 가지므로 대량 트래픽 발생 시 비용 모니터링이 필수적입니다.
* **인증서 비용**: AWS Certificate Manager(ACM)를 통해 발급받은 공인 인증서는 무료이지만, 외부에서 인증서를 구매하여 연동하거나 별도의 인증 관리 솔루션을 쓸 경우 추가 비용이 발생할 수 있습니다.
* **미삭제 주의**: 서비스와 마찬가지로 인그레스를 삭제하지 않고 클러스터만 제거하면 클라우드 로드밸런서가 남아 과금이 계속될 수 있으므로 `kubectl delete ingress`를 먼저 수행해야 합니다.

---
## 인그레스(Ingress)와 서비스(Service)의 비교 및 연동

쿠버네티스 네트워크를 설계할 때 **서비스**와 **인그레스**는 서로 대체재가 아닌, 상호 보완적인 관계입니다. 서비스가 클러스터 내부의 통신과 기본적인 외부 노출을 담당한다면, 인그레스는 그 상단에서 트래픽을 정교하게 제어하는 '지능형 게이트웨이' 역할을 수행합니다.

---

### 1. 계층적 차이 (L4 vs L7)

수강생들이 가장 직관적으로 이해할 수 있도록 네트워크 계층 모델(OSI 7 Layer) 관점에서 비교합니다.

| 구분 | 서비스 (Service) | 인그레스 (Ingress) |
| --- | --- | --- |
| **네트워크 계층** | **L4 (Transport Layer)** | **L7 (Application Layer)** |
| **식별 기준** | IP 주소 및 포트 번호 | 도메인(Host) 및 URL 경로(Path) |
| **주요 기능** | 부하 분산, 고정 진입점 제공 | 가상 호스팅, SSL 종단, 경로 기반 분기 |
| **비유** | 아파트의 '동/호수' (정확한 위치) | 아파트의 '안내 데스크' (방문 목적 확인 후 안내) |

---

### 2. 서비스와 인그레스의 연동 구조

인그레스는 단독으로 포드(Pod)에 트래픽을 보낼 수 없습니다. 반드시 **서비스를 거쳐야만** 포드에 도달할 수 있습니다.

**[트래픽 흐름]**

1. **클라이언트**: `http://my-it-class.com/api` 접속
2. **인그레스**: 요청을 수신하고 규칙 확인 (`/api` 경로는 `api-service`로 전달 결정)
3. **서비스**: 인그레스로부터 받은 트래픽을 `api-service`에 연결된 포드들로 분산
4. **포드**: 최종적으로 요청을 처리

---

### 3. 실무 환경에서의 구성 모델

강의 시 아래 두 가지 모델의 차이점을 강조해 주시면 좋습니다.

#### 모델 A: 서비스 단독 노출 (NodePort / LoadBalancer)

* **특징**: 각 서비스마다 고유의 IP/Port 또는 로드밸런서가 할당됨.
* **단점**: 서비스가 10개면 로드밸런서도 10개가 필요하여 비용이 급증하고, 도메인 관리가 복잡해짐.

#### 모델 B: 인그레스 통합 노출 (Ingress + ClusterIP)

* **특징**: 서비스는 `ClusterIP`로 숨기고, 인그레스 하나만 `LoadBalancer`로 노출함.
* **장점**: 로드밸런서 하나로 수십 개의 서비스를 통합 관리하여 **비용을 획기적으로 절감**하고 보안을 강화함.

---

### 4. 인그레스와 서비스의 YAML 관계도

두 오브젝트가 어떻게 이름을 통해 연결되는지 보여주는 핵심 코드입니다.

```yaml
# 1. 서비스 정의
apiVersion: v1
kind: Service
metadata:
  name: billing-svc         # 인그레스가 참조할 이름
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: billing-app
---
# 2. 인그레스 정의
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
spec:
  rules:
  - host: billing.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: billing-svc # 위 서비스 이름과 정확히 일치해야 함
            port:
              number: 80

```

---

### 💡 실무 전문가의 팁: 서비스 타입 선택

인그레스와 연동할 때 서비스의 타입(`spec.type`)은 무엇이 적당할까요? 외부 노출은 인그레스가 담당하므로 서비스는 클러스터 내부용인 **ClusterIP**면 충분합니다. 불필요하게 NodePort나 LoadBalancer를 사용할 필요가 없으므로 보안과 리소스 효율성 면에서 최적의 선택이 됩니다.

---

### ⚠️ 비용 및 유료 전환 주의사항

* **ALB/NLB 통합 과금**: AWS에서 인그레스를 통해 여러 서비스를 묶으면 로드밸런서 개수를 줄여 기본료를 아낄 수 있습니다. 하지만 **처리되는 트래픽 양(LcU)**에 따라 비용이 산정되므로, 미디어 스트리밍 등 고대역폭 서비스가 포함될 경우 비용이 예상보다 높을 수 있습니다.
* **공인 IP 비용**: AWS는 사용 중인 공인 IP에 대해서도 비용을 부과하므로, 인그레스를 통해 IP 사용 개수를 최소화하는 것이 경제적입니다.

---
맞습니다. **인그레스(Ingress)**는 주로 클러스터 **외부에서 내부로 들어오는 입구(L7 계층)**를 관리하는 관문 역할을 하며, 클러스터 내부 서비스들끼리의 통신과 로드밸런싱은 인그레스가 아닌 다른 주체가 담당합니다.

질문하신 내부 백엔드 서비스들 사이의 로드밸런싱은 **Service(ClusterIP)**와 그 배후에 있는 **kube-proxy**가 담당합니다.

---

## 1. 내부 로드밸런싱의 주인공: Service (ClusterIP)

클러스터 내부에서 프론트엔드 포드가 백엔드 포드를 호출할 때, 백엔드 포드들의 대표 주소인 `ClusterIP`로 요청을 보냅니다. 이때 실제로 부하를 분산해주는 메커니즘은 다음과 같습니다.

### 로드밸런싱의 흐름

1. **프론트엔드**: "백엔드 서비스(10.96.0.50)로 데이터 좀 보내줘!"
2. **노드의 커널(iptables/IPVS)**: 이 요청을 가로챕니다.
3. **kube-proxy**: API 서버로부터 받은 정보를 바탕으로 커널에 미리 로드밸런싱 규칙을 심어두었습니다.
4. **전달**: 커널이 살아있는 백엔드 포드들(`10.244.x.x`) 중 하나를 무작위 또는 라운드로빈 방식으로 선택하여 패킷을 보냅니다.

---

## 2. Ingress vs Service 로드밸런싱 비교

수강생들이 가장 많이 혼동하는 부분입니다. 아래 표로 명확히 구분해 주시면 좋습니다.

| 구분 | 인그레스 (Ingress) | 서비스 (ClusterIP) |
| --- | --- | --- |
| **위치** | 클러스터의 **최외곽 (Entry Point)** | 클러스터 **내부 (Internal)** |
| **계층** | **L7 (HTTP/HTTPS)** | **L4 (TCP/UDP)** |
| **핵심 역할** | 도메인/경로 기반 라우팅 (`/api`, `/web`) | 포드 그룹에 대한 고정 IP 및 부하 분산 |
| **통신 방향** | 외부 사용자 → 서비스 | 서비스 → 서비스 (동료끼리 통신) |
| **비유** | 회사의 **안내 데스크** (방문객 안내) | 부서 내의 **대표 번호** (내선 통신) |

---

## 3. 실제 사례로 보는 역할 분담

강사님의 서비스 시나리오(Frontend-Backend-DB)에 대입해 보겠습니다.

1. **Ingress**: 외부 사용자가 `example.com/api`로 접속하면 이를 **Backend-Svc**로 안내합니다.
2. **Service (ClusterIP)**:
* **Backend-Svc**: 인그레스에서 넘어온 요청을 실제 여러 개의 **Backend-Pod**들에게 골고루 나눠줍니다. (**내부 로드밸런싱**)
* **DB-Svc**: 백엔드가 DB를 호출할 때, 여러 개의 DB 복제본이 있다면 그중 하나로 연결해줍니다.





---

Next Step: 인그레스 컨트롤러(Nginx Ingress) 설치 및 경로 기반 라우팅 실습

실제로 하나의 대표 IP로 접속해서 경로(`/shop`, `/admin`)에 따라 서로 다른 서비스로 연결되는 **L7 라우팅 실습**을 교재에 추가해 볼까요? 혹은 **kube-proxy**의 동작 모드(iptables vs IPVS)의 성능 차이에 대해 다뤄볼까요?
