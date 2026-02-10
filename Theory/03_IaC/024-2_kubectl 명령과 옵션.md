
---

## kubectl 핵심 명령어 및 옵션 가이드

모든 예제에서 `[RESOURCE]`는 `pod`, `deployment(deploy)`, `service(svc)` 등으로 대체 가능합니다.

| 실무 예제 (Example) | 명령어 (Command) | 주요 옵션 (Options) | 용도 설명 |
| --- | --- | --- | --- |
| `kubectl apply -f app.yaml` | **apply** | `-f` (파일 지정) | 설정 파일을 읽어 리소스 생성 및 상태 업데이트 |
| `kubectl get pods -o wide` | **get** | `-o wide`, `-n`, `-A`, `-w` | 리소스 목록 조회 (상세 정보, 네임스페이스, 실시간 감시) |
| `kubectl describe pod my-pod` | **describe** | (리소스 명칭) | 특정 리소스의 상세 상태 및 **이벤트 로그** 확인 |
| `kubectl logs -f my-pod --tail=20` | **logs** | `-f`, `-c`, `--tail`, `-p` | 컨테이너 로그 확인 (실시간, 특정 컨테이너, 마지막 라인) |
| `kubectl exec -it my-pod -- bash` | **exec** | `-it` (터미널 접속) | 실행 중인 컨테이너 내부에 접속하여 명령어 실행 |
| `kubectl delete pod my-pod --force` | **delete** | `-f`, `--force`, `--grace-period=0` | 리소스 삭제 (강제 삭제 및 즉시 제거 옵션) |
| `kubectl run nginx --image=nginx` | **run** | `--image`, `--env`, `--port` | 간단한 테스트용 단일 포드 즉시 생성 및 실행 |
| `kubectl edit deploy my-deploy` | **edit** | (기본 에디터 실행) | 실행 중인 리소스의 YAML 설정을 실시간으로 직접 수정 |
| `kubectl cp ./file my-pod:/tmp` | **cp** | (Source) (Dest) | 로컬 파일 시스템과 컨테이너 간 파일 복사 |
| `kubectl port-forward my-pod 8080:80` | **port-forward** | (Local Port):(Pod Port) | 외부 노출 없이 로컬 포트를 포드 포트에 터널링 연결 |


쿠버네티스에서 특정 **Pod**를 직접 외부나 내부로 노출하고 싶을 때 `kubectl expose` 명령어를 사용합니다. 보통은 Deployment를 노출하지만, 개별 Pod 단위로도 서비스 엔드포인트를 생성할 수 있습니다.

---

## 1. 개별 Pod 노출 (kubectl expose pod)

실행 중인 특정 Pod를 대상으로 서비스를 생성하는 기본 명령어입니다.

```bash
# 특정 pod를 NodePort 방식으로 노출
kubectl expose pod <pod_name> --name=<service_name> --type=NodePort --port=80 --target-port=8080

```

* **--name**: 생성될 서비스의 이름입니다.
* **--type**: 노출 방식입니다. (`ClusterIP`, `NodePort`, `LoadBalancer`)
* **--port**: 서비스가 외부(클러스터 내부 혹은 외부)에 노출할 포트입니다.
* **--target-port**: Pod 내부의 컨테이너가 실제로 리스닝하고 있는 포트입니다.

---

## 2. 노출 방식별 차이점

| 타입 | 외부 접근 여부 | 특징 |
| --- | --- | --- |
| **ClusterIP** | 불가 | 클러스터 내부에서만 통신할 때 사용 (기본값) |
| **NodePort** | 가능 | 모든 노드의 특정 포트(30000-32767)를 통해 접근 |
| **LoadBalancer** | 가능 | 클라우드 제공업체의 로드밸런서를 생성하여 외부 IP 할당 |

---

## 3. 임시 노출: Port-Forward (권장)

교재 학습 환경이나 디버깅 시에는 `expose` 명령어로 고정된 서비스를 만드는 대신, **Port-Forward**를 사용하여 내 로컬 PC와 Pod를 임시로 연결하는 것이 훨씬 간편하고 안전합니다.

```bash
# 로컬의 8888 포트를 Pod의 80포트로 연결
kubectl port-forward pod/<pod_name> 8888:80

```

* 브라우저에서 `localhost:8888`로 접속하면 해당 Pod로 바로 연결됩니다.
* 명령어를 종료(`Ctrl+C`)하면 노출도 즉시 중단됩니다.

---

## 💡 전문가의 팁: 왜 Pod를 직접 expose하지 않나요?

실무에서는 개별 Pod를 `expose`하는 경우가 드뭅니다. 그 이유는 다음과 같습니다.

* **휘발성**: Pod가 재시작되어 이름이 바뀌면 기존 서비스와의 연결이 깨질 수 있습니다.
* **확장성**: Pod 하나만 노출하면 트래픽 부하 분산(Load Balancing) 기능을 활용할 수 없습니다.

> **해결책**: 항상 **Deployment**를 먼저 생성하고, 해당 Deployment를 `expose` 하세요. 그러면 Pod가 죽거나 새로 생성되어도 서비스는 변하지 않는 레이블을 통해 자동으로 새 Pod를 찾아 연결합니다.

> [!CAUTION] **비용 안내 (AWS 등 Public Cloud)**
> `type=LoadBalancer`를 사용하여 expose할 경우, AWS 기준 **ELB(Elastic Load Balancer) 사용료가 즉시 발생**합니다. 테스트 목적이라면 `NodePort`나 `port-forward`를 먼저 활용하는 것이 경제적입니다.

---

**Next Step:** Deployment와 Service를 연결하는 Label Selector 작동 원리

수정이 필요한 특정 구문이나 추가하고 싶은 내용이 있다면 알려주세요. 해당 부분만 즉시 보완하겠습니다.
---

## 상황별 명령어 조합 (강의 강조 포인트)

### 1. 배포 후 상태가 이상할 때 (Troubleshooting)

1. `kubectl get pods`: 전체적인 실행 상태(Running, Error, CrashLoop) 확인
2. `kubectl describe pod [NAME]`: 왜 에러가 났는지 **Events** 섹션 확인 (이미지 오류, 스케줄링 오류 등)
3. `kubectl logs [NAME] -p`: 컨테이너 내부 애플리케이션 로그(이전 로그 포함) 확인

### 2. 설정 변경 및 관리

* **실시간 반영**: `kubectl apply -f [FILE]` (권장)
* **긴급 수정**: `kubectl edit [RESOURCE] [NAME]`
* **리소스 확장**: `kubectl scale deployment [NAME] --replicas=5`

### 3. 클러스터 환경 설정

* `kubectl config view`: 현재 접속 정보 확인
* `kubectl config use-context [NAME]`: 접속할 클러스터 전환

---

Next Step: kubectl alias 및 자동 완성(Autocomplete) 설정

매번 `kubectl`을 타이핑하기 번거로우므로, `k`로 줄여 쓰고 `Tab` 키로 옵션을 자동 완성하는 설정을 교재에 추가해 볼까요?
