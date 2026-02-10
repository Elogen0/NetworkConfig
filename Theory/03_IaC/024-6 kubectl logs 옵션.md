쿠버네티스에서 컨테이너의 상태를 파악하고 문제를 해결할 때 가장 많이 사용하는 명령어인 `kubectl logs`의 주요 옵션들을 정리해 드립니다. 강의 교재에 바로 활용하실 수 있도록 실무에서 자주 쓰이는 패턴 위주로 구성했습니다.

---

## kubectl logs 주요 옵션 및 사용법

### 1. 기본 사용법

특정 포드(Pod)의 로그를 확인하는 가장 기본적인 형태입니다.

```bash
kubectl logs <pod-name>

```

### 2. 실시간 로그 모니터링 (`-f`, `--follow`)

로그가 기록되는 것을 실시간으로 계속 확인하고 싶을 때 사용합니다. (Linux의 `tail -f`와 동일)

```bash
kubectl logs -f <pod-name>

```

### 3. 멀티 컨테이너 포드 로그 (`-c`, `--container`)

하나의 포드 안에 여러 개의 컨테이너가 실행 중인 경우, 특정 컨테이너를 지정해야 합니다.

```bash
# pod-name 안에 있는 nginx 컨테이너 로그 확인
kubectl logs <pod-name> -c nginx

```

### 4. 출력 라인 수 제한 (`--tail`)

로그의 양이 너무 많을 때 마지막 N줄만 출력합니다.

```bash
# 마지막 20줄만 확인
kubectl logs --tail=20 <pod-name>

```

### 5. 특정 시간 범위 로그 (`--since`, `--since-time`)

특정 시간 동안 발생한 로그만 필터링할 때 유용합니다.

```bash
# 최근 1시간 동안의 로그만 확인
kubectl logs --since=1h <pod-name>

# 특정 시점(RFC3339 형식) 이후의 로그 확인
kubectl logs --since-time=2026-02-01T15:00:00Z <pod-name>

```

### 6. 이전 컨테이너 로그 확인 (`-p`, `--previous`)

컨테이너가 비정상적으로 종료되어 재시작된 경우, **종료되기 직전(Crash 발생 시점)**의 로그를 볼 수 있는 매우 중요한 옵션입니다.

```bash
kubectl logs -p <pod-name>

```

### 7. 타임스탬프 표시 (`--timestamps`)

로그의 각 줄 앞에 생성된 시간을 함께 표시합니다.

```bash
kubectl logs --timestamps <pod-name>

```

### 8. 라벨 셀렉터 활용 (`-l`)

특정 라벨을 가진 모든 포드의 로그를 한꺼번에 확인합니다.

```bash
# app=frontend 라벨을 가진 모든 포드 로그 확인
kubectl logs -l app=frontend

```

---

## 실무 활용 팁 요약 테이블

| 상황 | 추천 명령어 조합 |
| --- | --- |
| **실시간 장애 모니터링** | `kubectl logs -f <pod-name>` |
| **포드 재시작 원인 파악** | `kubectl logs -p <pod-name>` |
| **최근 발생한 에러만 필터링** | `kubectl logs --tail=50 <pod-name>` |
| **여러 컨테이너 중 선택** | `kubectl logs <pod-name> -c <container-name>` |
| **특정 시점 로그 추적** | `kubectl logs --since=30m --timestamps <pod-name>` |

---

Next Step: 쿠버네티스 이벤트(Events) 확인 및 트러블슈팅

로그만으로 원인 파악이 힘들 때 사용하는 `kubectl describe`나 `kubectl get events`를 활용한 장애 진단법에 대해서도 알아볼까요?
