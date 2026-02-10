<img width="592" height="491" alt="image" src="https://github.com/user-attachments/assets/b3976aed-b341-4be5-9162-1d98cd3ecb9a" />


## Pod의 개념과 역할

Kubernetes에서 **Pod(파드)**는 생성하고 관리할 수 있는 **배포 가능한 가장 작은 컴퓨팅 단위**입니다. 하나 이상의 컨테이너(Docker 등)를 그룹화한 것으로, Pod 내의 컨테이너들은 네트워크와 스토리지를 공유하며 밀접하게 결합되어 동작합니다.

### 1. Pod의 주요 특징

* **원자적 단위:** Kubernetes는 컨테이너를 개별적으로 관리하지 않고 Pod 단위로 스케줄링하고 관리합니다.
* **공유 네트워크 (IP):** Pod 내의 모든 컨테이너는 동일한 네트워크 네임스페이스를 공유합니다. 따라서 컨테이너끼리 `localhost`를 통해 통신할 수 있으며, 동일한 포트 번호를 사용할 수 없습니다.
* **공유 스토리지 (Volume):** Pod 설정에 정의된 볼륨을 Pod 내의 모든 컨테이너가 마운트하여 데이터를 공유할 수 있습니다.
* **휘발성 (Ephemeral):** Pod는 영구적인 존재가 아닙니다. 노드 장애나 스케줄링 정책에 의해 언제든 삭제되고 새로 생성될 수 있으며, 이때 IP 주소는 변경됩니다.

---
<img width="837" height="556" alt="image" src="https://github.com/user-attachments/assets/753ea8c6-5c8a-4d23-99a1-d397985b3313" />



### 2. Pod의 구성 방식

Pod는 크게 두 가지 형태로 구성됩니다.

| 구성 방식 | 설명 | 주요 사용 사례 |
| --- | --- | --- |
| **Single-Container Pod** | 가장 일반적인 형태이며, 하나의 Pod에 하나의 컨테이너만 실행합니다. | 일반적인 웹 서버, API 서버 등 |
| **Multi-Container Pod** | 주 컨테이너를 보조하는 '사이드카(Sidecar)' 컨테이너를 함께 실행합니다. | 로그 수집기, 프록시 설정, 데이터 동기화 등 |

> **실전 팁:** 한 Pod에 여러 컨테이너를 넣는 것은 두 컨테이너가 반드시 **동일한 생명주기**를 가져야 하고, 리소스를 강하게 공유해야 할 때만 사용하세요. 단순히 통신이 필요하다면 별개의 Pod로 분리하고 Service를 통해 연결하는 것이 확장성 면에서 유리합니다.

---

### 3. Pod 생명주기 (Lifecycle)

Pod의 상태는 `status.phase` 필드를 통해 확인할 수 있습니다.

1. **Pending:** Pod가 클러스터에 승인되었지만, 하나 이상의 컨테이너가 실행 준비 중인 상태(이미지 다운로드 등).
2. **Running:** Pod가 노드에 할당되었고, 모든 컨테이너가 생성되었으며 적어도 하나 이상의 컨테이너가 실행 중인 상태.
3. **Succeeded:** Pod 내의 모든 컨테이너가 성공적으로 종료되었으며 재시작되지 않는 상태 (주로 Job 객체에서 발생).
4. **Failed:** Pod 내의 모든 컨테이너가 종료되었으나, 적어도 하나 이상의 컨테이너가 실패로 종료된 상태.
5. **Unknown:** Pod의 상태를 확인할 수 없는 경우 (주로 노드와의 통신 단절).

---

### 4. YAML 정의 예시

Pod를 생성하기 위한 표준 `pod.yaml` 형식입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-webapp
  labels:
    app: web
spec:
  containers:
  - name: main-container
    image: nginx:1.25
    ports:
    - containerPort: 80
  - name: sidecar-logger
    image: busybox
    command: ["sh", "-c", "while true; do date; sleep 10; done"]

```

---

### 💡 자주 오해하는 부분

* **"Pod는 고정된 IP를 가진다?"**
아니요. Pod가 재생성되면 IP는 매번 바뀝니다. 이를 해결하기 위해 **Service**라는 객체를 사용하여 고정된 진입점(VIP)을 제공해야 합니다.
* **"직접 Pod를 생성해도 될까?"**
테스트 목적이 아니라면 `kind: Pod`를 직접 생성하는 것은 권장되지 않습니다. 대신 **Deployment**나 **StatefulSet** 같은 컨트롤러를 사용하여 복제본 관리 및 자동 복구(Self-healing) 기능을 활용해야 합니다.

---

요청하신 대로 교재 구성에 맞춰 내용을 정리했습니다.

**Next Step:** Pod의 네트워킹 및 통신 모델 (Service 연결)
