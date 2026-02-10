쿠버네티스(Kubernetes)에서 **emptyDir**은 파드(Pod)가 특정 노드에 할당될 때 처음 생성되며, 해당 파드가 해당 노드에서 실행되는 동안에만 존재하는 임시 볼륨입니다.

---

## 1. emptyDir의 개념 및 특징

* **생명주기 동기화:** 파드와 함께 생성되고, 파드가 노드에서 삭제되면 `emptyDir`의 데이터도 **영구적으로 삭제**됩니다. (컨테이너가 단순히 재시작되는 것으로는 데이터가 삭제되지 않습니다.)
* **임시 저장 공간:** 이름 그대로 처음에는 비어 있는 상태(`empty`)로 제공됩니다.
* **로컬 저장소 활용:** 기본적으로 노드의 디스크(HDD/SSD) 공간을 사용하지만, 설정을 통해 메모리(RAM)를 사용할 수도 있습니다.

---

## 2. 주요 사용 사례 (Use Cases)

`emptyDir`은 영구 저장이 필요 없는 임시 작업 공간이 필요할 때 매우 유용합니다.

1. **스크래치 공간 (Scratch Space):** 대규모 데이터 처리나 정렬 작업 시 발생하는 임시 파일을 저장할 때 사용합니다.
2. **체크포인트 저장:** 긴 실행 시간이 필요한 작업에서 중단 시점을 기록하고, 컨테이너 재시작 시 해당 지점부터 다시 시작할 수 있게 돕습니다.
3. **사이드카 컨테이너 간 데이터 공유:**
* **공유 예시:** 하나의 파드 안에 `웹 서버` 컨테이너와 `콘텐츠 매니저` 컨테이너가 있을 때, 매니저가 외부에서 데이터를 받아 `emptyDir`에 저장하면 웹 서버가 이를 즉시 서비스할 수 있습니다.


4. **메모리 기반 고속 저장소:** `medium: Memory` 설정을 통해 램디스크(RAM 디스크)처럼 활용하여 고속 읽기/쓰기 작업을 수행합니다.

---

## 3. 설정 예시 (YAML)

다음은 두 개의 컨테이너가 하나의 `emptyDir` 볼륨을 공유하는 예시입니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-example
spec:
  containers:
  - name: producer
    image: alpine
    volumeMounts:
    - mountPath: /cache
      name: shared-data
    command: ["/bin/sh", "-c", "while true; do date >> /cache/index.html; sleep 10; done"]

  - name: consumer
    image: nginx
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: shared-data

  volumes:
  - name: shared-data
    emptyDir:
      # RAM을 사용하고 싶을 경우 아래 주석 해제 (생략 시 디스크 사용)
      # medium: Memory

```

---

## 4. hostPath와 emptyDir 비교

| 항목 | emptyDir | hostPath |
| --- | --- | --- |
| **초기 상태** | 항상 비어 있음 | 노드의 기존 파일/디렉터리 포함 가능 |
| **영속성** | 파드 삭제 시 데이터 삭제 | 파드 삭제 후에도 노드에 데이터 유지 |
| **용도** | 임시 작업, 컨테이너 간 공유 | 시스템 로그 수집, 로컬 장치 접근 |
| **안전성** | 상대적으로 안전함 | 노드 파일 시스템 노출로 보안 위험 존재 |

---

## 5. 실전 팁: 리소스 제한

`emptyDir`은 노드의 디스크 공간을 사용하므로, 무분별한 파일 생성으로 노드 전체의 디스크가 가득 차는 현상(Disk Pressure)을 방지해야 합니다. 이를 위해 `sizeLimit`을 설정하여 사용할 수 있는 최대 용량을 제한하는 것이 권장됩니다.

```yaml
volumes:
- name: shared-data
  emptyDir:
    sizeLimit: "500Mi"

```

---

## Next Step:

**PersistentVolume(PV)과 PersistentVolumeClaim(PVC)의 개념 및 차이점 분석**
