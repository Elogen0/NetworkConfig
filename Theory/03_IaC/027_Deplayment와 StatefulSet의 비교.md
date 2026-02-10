## Deployment vs. StatefulSet 상세 비교

쿠버네티스에서 애플리케이션을 배포할 때 가장 많이 사용하는 두 가지 워크로드 리소스인 **Deployment**와 **StatefulSet**은 애플리케이션의 '상태(State)' 유무에 따라 설계 철학이 완전히 다릅니다. 각 리소스의 특징과 실무적 차이점을 분석합니다.

---

### 1. 핵심 특징 비교 테이블

| 구분 | Deployment (Stateless) | StatefulSet (Stateful) |
| --- | --- | --- |
| **적합한 서비스** | 웹 서버, API 서버 (상태 비저장) | 데이터베이스, 분산 파일 시스템, 메세지 큐 |
| **포드 이름** | 무작위 해시값 (`app-67487-abcde`) | 순차적 번호 부여 (`app-0`, `app-1`, `app-2`) |
| **네트워크 식별자** | 가변적 (서비스 VIP를 통해 접근) | 고정적 (Headless Service를 통한 개별 DNS) |
| **스토리지 방식** | 보통 공유 스토리지를 쓰거나 없음 | 각 포드마다 전용 볼륨 부여 (1:1 매핑) |
| **배포/확장 순서** | 병렬적 (순서 상관없이 생성/삭제) | 순차적 (0번 완료 후 1번 시작) |
| **데이터 유지** | 포드 삭제 시 데이터 유실 위험 높음 | 포드 재시작/삭제 시에도 고유 볼륨 유지 |

---

### 2. 포드 식별자 및 네트워크 주소 체계

#### Deployment

* **익명성:** 모든 포드는 동일한 복제본으로 취급됩니다. 포드가 죽고 새로 생성되면 이름과 IP가 완전히 바뀝니다.
* **접근 방식:** 개별 포드에 직접 접근하기보다는 서비스(Service)가 제공하는 단일 가상 IP를 통해 트래픽을 분산 처리합니다.

#### StatefulSet

* **고유성:** 각 포드는 고유한 인덱스(0, 1, 2...)를 가지며, 이 이름은 포드가 재배포되어도 영구적으로 유지됩니다.
* **Headless Service 연결:** 스테이트풀셋은 반드시 헤드리스 서비스와 연결되어야 합니다. 이를 통해 `mariadb-0.mariadb-svc`와 같은 고정된 내부 DNS 주소를 가집니다.

---

### 3. 스토리지 운영 방식 (Persistence)

#### Deployment: 공용 혹은 비영구적

* 대부분의 Deployment는 로컬 스토리지를 사용하지 않거나, 다수의 포드가 읽기 전용으로 공유하는 데이터를 사용합니다. 포드가 교체되면 이전 포드가 쓰던 로컬 데이터는 사라집니다.

#### StatefulSet: 전용 및 영구적

* **VolumeClaimTemplate:** 이 설정을 통해 각 포드가 생성될 때마다 개별적인 PVC(Persistent Volume Claim)가 자동으로 생성됩니다.
* **데이터 보존:** `app-0` 포드가 장애로 인해 다른 노드에서 다시 생성되더라도, 기존에 `app-0`이 사용하던 전용 볼륨(`pvc-app-0`)을 다시 찾아가서 마운트합니다.

---

### 4. 배포 및 업데이트 전략

* **Deployment:** 기본적으로 `RollingUpdate`를 사용하며, 여러 개의 포드를 동시에 죽이고 생성하는 등 병렬적인 처리가 가능합니다. 빠르게 확장(Scaling)하는 데 유리합니다.
* **StatefulSet:** 데이터의 일관성과 클러스터 멤버 간의 합의(Consensus)가 중요하므로, 반드시 한 번에 하나씩 순차적으로 작업이 진행됩니다. 0번 포드가 완전히 서비스 가능 상태(`Ready`)가 되어야만 1번 포드 작업으로 넘어갑니다.

---

### 💡 AWS 환경 운영 및 과금 가이드

* **EBS 볼륨 비용:** **StatefulSet**은 포드마다 개별 EBS 볼륨을 생성하므로, Deployment 대비 **EBS(Elastic Block Store) 비용**이 복제본 개수에 비례하여 정직하게 증가합니다.
* **데이터 전송 요금 (Inter-AZ):** 고가용성을 위해 DB 포드들이 서로 다른 가용 영역(AZ)에 배포된 경우, 마스터와 슬레이브 간의 데이터 복제 과정에서 발생하는 **[Amazon EC2 Data Transfer](https://www.google.com/search?q=https://aws.amazon.com/ec2/pricing/on-demand/%23Data_Transfer)** 비용을 반드시 고려해야 합니다.
* **관리형 서비스 전환 고려:** 직접 StatefulSet으로 DB를 구축하는 것과 **[AWS RDS](https://aws.amazon.com/rds/)** 또는 **[Amazon EKS Managed Add-ons](https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html)**를 사용하는 것의 비용 및 운영 공수를 비교하여 선택하십시오. 직접 운영 시 백업, 패치 등의 관리 비용이 추가됩니다.

---

Next Step: Persistent Volume Claim(PVC)의 생명 주기와 Reclaim Policy

---

Deployment와 StatefulSet의 구조적 차이를 이해하면, 애플리케이션의 성격에 맞는 최적의 배포 전략을 수립할 수 있습니다. 특히 데이터의 중요도가 높은 서비스일수록 StatefulSet의 고유 식별자와 스토리지 보존 기능을 적극 활용해야 합니다.

Deployment 특징, StatefulSet 특징, 스토리지 매핑, AWS 비용 관리
