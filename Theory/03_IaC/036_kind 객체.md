쿠버네티스에서 가장 자주 사용되는 주요 객체(Object)별 YAML 예시 10가지입니다. 각 설정의 의미를 파악할 수 있도록 행 단위로 주석을 추가하였습니다.

---

### **1. Pod (파드)**

가장 기본적인 배포 단위로, 하나 이상의 컨테이너를 포함합니다.

```yaml
apiVersion: v1              # API 버전
kind: Pod                   # 객체 종류
metadata:
  name: nginx-pod           # 파드 이름
spec:
  containers:               # 컨테이너 설정
  - name: nginx             # 컨테이너 이름
    image: nginx:latest     # 사용할 이미지
    ports:
    - containerPort: 80     # 컨테이너가 사용할 포트

```

### **2. Deployment (디플로이먼트)**

파드의 개수를 유지하고 배포 전략(롤링 업데이트 등)을 관리합니다.

```yaml
apiVersion: apps/v1         # Deployment의 API 버전
kind: Deployment            # 객체 종류
metadata:
  name: web-deploy          # 디플로이먼트 이름
spec:
  replicas: 3               # 유지할 파드 개수
  selector:                 # 어떤 파드를 관리할지 선택하는 라벨
    matchLabels:
      app: web
  template:                 # 생성될 파드의 템플릿
    metadata:
      labels:
        app: web            # 파드에 부여될 라벨
    spec:
      containers:
      - name: web-app
        image: nginx:1.21

```

### **3. Service - ClusterIP (서비스)**

클러스터 내부 통신용 고정 IP를 제공합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: internal-svc        # 서비스 이름
spec:
  selector:                 # 대상 파드를 지정하는 라벨
    app: web
  ports:
  - protocol: TCP
    port: 80                # 서비스가 노출할 포트
    targetPort: 80          # 파드로 전달할 포트
  type: ClusterIP           # 클러스터 내부 전용 (기본값)

```

### **4. Service - LoadBalancer (서비스)**

외부에서 접속 가능한 로드밸런서를 생성합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-svc
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer        # 외부 로드밸런서 할당 (AWS 사용 시 비용 발생)

```

### **5. Namespace (네임스페이스)**

클러스터 내의 자원을 논리적으로 분리합니다.

```yaml
apiVersion: v1
kind: Namespace             # 객체 종류
metadata:
  name: dev-team            # 생성할 네임스페이스 이름

```

### **6. ConfigMap (컨피그맵)**

설정 데이터를 키-값 쌍으로 저장하여 컨테이너에 전달합니다.

```yaml
apiVersion: v1
kind: ConfigMap             # 객체 종류
metadata:
  name: app-config
data:
  DB_URL: "localhost:3306"  # 설정 데이터 (Key-Value)
  DEBUG_MODE: "true"

```

### **7. Secret (시크릿)**

비밀번호, 키 등 보안이 필요한 데이터를 Base64로 인코딩하여 저장합니다.

```yaml
apiVersion: v1
kind: Secret                # 객체 종류
metadata:
  name: db-secret
type: Opaque                # 일반적인 보안 데이터 타입
data:
  password: YWRtaW4xMjM0    # 'admin1234'를 Base64 인코딩한 값

```

### **8. PersistentVolumeClaim (PVC)**

스토리지 자원을 사용하기 위해 요청합니다.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim # 객체 종류
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce         # 단일 노드에서 읽기/쓰기 가능
  resources:
    requests:
      storage: 1Gi          # 요청할 스토리지 용량 (AWS 사용 시 비용 발생)

```

### **9. Ingress (인그레스)**

외부 HTTP/HTTPS 트래픽을 서비스로 라우팅합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress               # 객체 종류
metadata:
  name: web-ingress
spec:
  rules:                    # 라우팅 규칙
  - host: myapp.com         # 도메인 설정
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: internal-svc
            port:
              number: 80

```

### **10. Job (잡)**

일회성 작업을 실행하고 완료되면 종료합니다.

```yaml
apiVersion: batch/v1
kind: Job                   # 객체 종류
metadata:
  name: batch-job
spec:
  template:
    spec:
      containers:
      - name: worker
        image: perl
        command: ["echo", "Done!"] # 실행할 명령어
      restartPolicy: Never   # 작업 실패 시 재시작 정책

```

---

### **핵심 설명 및 주의사항**

1. **apiVersion**: 각 객체마다 사용하는 API 그룹과 버전이 다릅니다. (예: Deployment는 `apps/v1`, Job은 `batch/v1`)
2. **Selector와 Labels**: 쿠버네티스에서 객체 간의 연결(Service가 특정 Pod를 찾는 등)은 이름을 직접 쓰지 않고 **라벨(Label)**을 통해 유연하게 연결됩니다.
3. **유료 서비스 안내**:
* **LoadBalancer**: `type: LoadBalancer`를 AWS 환경에서 실행하면 **Classic Load Balancer 또는 NLB**가 자동으로 생성되어 시간당 비용이 청구됩니다.
* **Storage (PVC)**: PVC를 생성하여 실제 데이터가 저장되는 볼륨(EBS 등)이 할당되면 **스토리지 용량에 따른 월별 비용**이 발생합니다.



**Next Step: 쿠버네티스 Label과 Selector의 동작 원리**

* 각 YAML 파일을 `kubectl apply -f [파일명].yaml` 명령어로 실행하여 실제 객체가 생성되는지 확인할 수 있습니다.
* 깃허브에 바로 업로드하여 사용할 수 있도록 표준 포맷을 준수하였습니다.
