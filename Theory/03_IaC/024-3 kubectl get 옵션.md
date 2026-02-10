## kubectl get 주요 옵션 및 사용법

### 1. 출력 형식 지정 (`-o`, `--output`)

가장 많이 사용되는 옵션으로, 정보를 보여주는 방식을 결정합니다.

* **`-o wide`**: 노드 IP, 이미지 이름 등 더 상세한 정보를 표 형식으로 출력합니다.
* **`-o yaml`**: 리스트나 특정 리소스의 전체 설정을 YAML 형식으로 보여줍니다. (백업 및 설정 확인용)
* **`-o json`**: 프로그래밍 처리가 용이하도록 JSON 형식으로 출력합니다.
* **`-o name`**: 리소스의 이름만 출력합니다 (스크립트 활용 시 유용).

```bash
# 포드 정보를 상세하게 조회 (IP, 노드명 포함)
kubectl get pods -o wide

# 특정 서비스 설정을 YAML로 확인
kubectl get svc <service-name> -o yaml

```

### 2. 특정 네임스페이스 및 전체 조회 (`-n`, `-A`)

* **`-n <namespace>`**: 특정 네임스페이스에 속한 리소스만 조회합니다.
* **`-A` (또는 `--all-namespaces`)**: 클러스터 내의 모든 네임스페이스에 있는 리소스를 한꺼번에 조회합니다.

```bash
# 모든 네임스페이스의 포드 상태 확인
kubectl get pods -A

```

### 3. 라벨 필터링 (`-l`, `--selector`)

특정 라벨(Label)이 붙은 리소스만 골라서 출력합니다.

```bash
# app=nginx 라벨을 가진 포드만 조회
kubectl get pods -l app=nginx

# env가 production이 아닌 리소스만 조회
kubectl get pods -l 'env!=production'

```

### 4. 실시간 감시 (`-w`, `--watch`)

리소스의 상태 변화(예: Pending -> Running)를 실시간으로 화면에 출력합니다.

```bash
kubectl get pods -w

```

### 5. 필드 선택기 (`--field-selector`)

라벨이 아닌 리소스의 특정 상태 값을 기준으로 필터링합니다.

```bash
# 현재 실행 중인(Running) 포드만 조회
kubectl get pods --field-selector status.phase=Running

# 특정 노드에 배치된 포드만 조회
kubectl get pods --field-selector spec.nodeName=worker-node01

```

### 6. 커스텀 컬럼 출력 (`-o custom-columns`)

사용자가 보고 싶은 항목만 골라서 표 형태로 만듭니다.

```bash
# 포드 이름과 상태값만 추출하여 출력
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

```

---

## 리소스별 조회 예시 테이블

| 대상 | 명령어 | 비고 |
| --- | --- | --- |
| **전체 리소스** | `kubectl get all` | 현재 네임스페이스의 주요 리소스 요약 |
| **노드 상태** | `kubectl get nodes` | 클러스터 구성 노드 확인 |
| **서비스/엔드포인트** | `kubectl get svc,ep` | 서비스와 연결된 IP 확인 |
| **설정 정보** | `kubectl get cm,secret` | ConfigMap과 Secret 리스트 조회 |
| **리소스 정의(CRD)** | `kubectl get crd` | 사용자 정의 리소스 확인 |
