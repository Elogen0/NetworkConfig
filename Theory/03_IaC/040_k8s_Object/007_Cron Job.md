## CronJob: 주기적인 작업의 예약 관리

**크론잡(CronJob)**은 리눅스의 `cron`과 마찬가지로, 정해진 시간에 반복적으로 **잡(Job)**을 생성하여 실행하는 컨트롤러입니다. 데이터베이스 백업, 리포트 생성, 정기적인 데이터 동기화 등 스케줄링이 필요한 작업에 필수적입니다.

---

### 1. 크론잡(CronJob)의 작동 원리

크론잡은 사용자가 지정한 스케줄에 따라 주기적으로 잡 객체를 생성합니다. 생성된 잡은 앞서 배운 대로 파드를 생성하여 실제 작업을 수행하고, 작업이 끝나면 파드를 종료합니다.

* **관계도:** CronJob → Job → Pod
* **스케줄링 형식:** `* * * * *` (분 시 일 월 요일) 형식을 사용합니다.

---

### 2. CronJob YAML 정의서 및 상세 분석

매일 새벽 2시에 백업 작업을 수행하는 가상의 크론잡 예시입니다.

```yaml
apiVersion: batch/v1                                  
kind: CronJob 
metadata: 
  name: daily-backup-cronjob 
spec: 
  schedule: "0 2 * * * "                    # 매일 02:00 (새벽 2시)에 실행하도록 설정
  jobTemplate:                             # 실행될 잡의 템플릿 (Job 설정과 동일)
    spec:
      template:
        spec:
          containers:
            - name: backup-worker
              image: busybox                                         # 예제용 가벼운 이미지
              command: ["/bin/sh", "-c", "echo 'Backup started'; sleep 30; echo 'Backup finished'"]
          restartPolicy: OnFailure # 실패 시에만 재시작
  successfulJobsHistoryLimit: 3 # 성공한 작업의 기록을 최대 3개까지만 유지
  failedJobsHistoryLimit: 1 # 실패한 작업의 기록을 최대 1개까지만 유지
  concurrencyPolicy: Forbid # 이전 작업이 끝나지 않았으면 다음 작업 실행 금지

```

---

### 3. 주요 설정 옵션 상세 설명

* **schedule:** 크론 표현식을 사용하여 주기를 설정합니다.
* **concurrencyPolicy (동시성 정책):**
* `Allow` (기본값): 이전 작업이 안 끝났어도 새 작업을 실행합니다.
* `Forbid`: 이전 작업이 실행 중이면 다음 차례의 작업을 건너뜁니다. (중복 실행 방지)
* `Replace`: 이전 작업을 종료하고 새 작업을 시작합니다.


* **startingDeadlineSeconds:** 설정된 시간에 작업이 시작되지 못했을 때, 몇 초까지 기다렸다가 실행할지 결정합니다.
* **HistoryLimit:** 완료된 작업 기록을 무한정 남기면 클러스터 자원을 낭비하므로, 성공/실패 기록 보관 개수를 지정하는 것이 좋습니다.

---

### 4. 실행 및 상태 확인

**Step 1: 크론잡 실행**

```bash
kubectl apply -f cronjob.yaml

```

**Step 2: 상태 점검**

```bash
kubectl get cronjob
# SCHEDULE과 마지막 실행 시간(LAST SCHEDULE)을 확인할 수 있습니다.

```

**Step 3: 생성된 잡 확인**

```bash
kubectl get job
# 크론잡에 의해 이름이 랜덤하게 생성된 Job들을 확인할 수 있습니다.

```

---

### 5. AWS 환경에서의 유료 서비스 및 비용 관리

* **서버리스 아키텍처 결합:** EKS를 사용 중이라면 크론잡을 **AWS Fargate**에서 실행하도록 설정할 수 있습니다. 이 경우 작업을 수행하는 동안만 초 단위로 과금되므로, 24시간 켜져 있는 EC2 노드 비용을 아낄 수 있습니다.
* **컴퓨팅 최적화:** 배치 작업의 특성상 리소스 점유가 일시적으로 급증할 수 있습니다. `requests`와 `limits`를 명확히 설정하여 다른 실시간 서비스 파드에 영향을 주지 않도록 설계해야 합니다.
* **주의사항:** 크론잡이 생성하는 파드가 짧은 시간에 반복적으로 생성/삭제되면 **CloudWatch Logs** 등에 로그 보관 비용이 발생할 수 있습니다. 필요한 로그만 남기도록 로그 레벨을 조정하는 것이 좋습니다.

---

**Next Step: Taints와 Tolerations를 이용한 노드 스케줄링**

* 특정 노드에만 파드를 배치하거나, 특정 노드에 배포되지 않도록 차단하는 고급 기법을 배워볼까요?

---
