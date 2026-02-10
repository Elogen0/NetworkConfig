## 데몬셋(DaemonSet)의 이해와 활용

**데몬셋(DaemonSet)**은 클러스터의 **모든(또는 특정) 노드에 개별적으로 Pod를 하나씩 반드시 실행**하도록 보장하는 컨트롤러입니다. 노드가 클러스터에 추가되면 해당 노드에 자동으로 Pod를 실행하고, 노드가 삭제되면 해당 Pod도 함께 정리되는 특징을 가집니다.

---

### 1. 데몬셋의 주요 용도

애플리케이션 서비스보다는 **인프라 관리 및 보조 시스템**을 구축할 때 주로 사용됩니다.

* **로그 수집**: 모든 노드에서 발생하는 로그를 수집하는 에이전트 실행 (예: Fluentd, Logstash)
* **모니터링**: 노드의 상태를 수집하여 전달하는 에이전트 실행 (예: Prometheus Node Exporter)
* **네트워크 플러그인**: 클러스터 네트워크 구성을 위한 데몬 실행 (예: Calico, Flannel)
* **스토리지**: 각 노드에 공유 스토리지를 제공하는 데몬 실행 (예: GlusterFS, Ceph)

---

### 2. 데몬셋 vs 디플로이먼트 비교

| 구분 | 디플로이먼트 (Deployment) | 데몬셋 (DaemonSet) |
| --- | --- | --- |
| **목적** | 일반적인 서비스(웹 서버, API 등) 배포 | 인프라 에이전트 및 관리 도구 배포 |
| **배치 방식** | 스케줄러가 리소스 상황에 맞춰 노드 선택 | **각 노드마다 1개씩** 고정 배치 |
| **수량 조절** | `replicas` 필드로 개수 조절 | 노드의 개수에 따라 자동으로 조절 |
| **용도** | 상태가 없는(Stateless) 앱 | 노드 단위 작업 (Logging, Monitoring) |

---

### 3. 데몬셋 YAML 예시 (로그 수집 에이전트)

```yaml
apiVersion: apps/v1
kind: DaemonSet                  # 리소스 종류가 데몬셋임을 명시
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system         # 주로 시스템 관리용이므로 kube-system 네임스페이스 사용
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:               # 마스터 노드에도 실행하고 싶을 때 테인트 허용 설정
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2

```

---

### 4. 특정 노드에만 배포하기 (nodeSelector)

모든 노드가 아닌 GPU가 있는 노드나 SSD가 있는 노드에만 데몬셋을 실행하고 싶을 때 사용합니다.

```yaml
spec:
  template:
    spec:
      nodeSelector:              # 특정 레이블이 붙은 노드에만 실행
        hardware: gpu

```

---

### 💡 실무 전문가의 팁: 마스터 노드 실행 여부

기본적으로 쿠버네티스 마스터 노드는 **테인트(Taint)** 설정 때문에 일반적인 Pod가 생성되지 않습니다. 하지만 클러스터 전체 로그 수집을 위해 마스터 노드에도 데몬셋을 띄워야 한다면, YAML 설정에 `tolerations`를 추가하여 마스터 노드의 접근 거부 설정을 통과해야 함을 수강생들에게 강조해 주세요.

---

### ⚠️ 비용 및 유료 전환 주의사항

* **리소스 누적 비용**: 데몬셋은 노드가 추가될 때마다 자동으로 실행되므로, **노드 개수에 비례하여 전체 클러스터 리소스(CPU/Mem) 사용량이 정비례로 증가**합니다. 고사양의 에이전트를 데몬셋으로 띄우면 노드 증설 시마다 예상보다 큰 비용이 발생할 수 있습니다.
* **AWS CloudWatch**: 데몬셋을 통해 로그를 수집하여 AWS CloudWatch로 전송할 경우, 전송되는 **데이터의 용량(GB) 및 로그 저장 기간**에 따라 상당한 AWS 과금이 발생할 수 있으니 필터링 설정을 권장합니다.

---

**Next Step: 특정 노드 그룹 필터링을 위한 Node Affinity 설정**

Next Step: **nodeAffinity 활용법** | **데몬셋 업데이트 전략(OnDelete vs RollingUpdate)** | **테인트와 톨러레이션 상세 분석** Would you like me to: **강의 교재용 데몬셋 실습 문제 작성**, **모니터링 에이전트 배포 예제 구성**?
