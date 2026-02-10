`kubelet`은 각 노드에서 실행되는 핵심 에이전트로, `kubectl`이나 `kubeadm`과 달리 사용자가 직접 명령어를 입력하기보다는 **시스템 서비스(systemd)** 형태로 관리됩니다. 하지만 트러블슈팅이나 커스텀 설정 시 필요한 주요 옵션들을 예제와 함께 정리해 드립니다.

---

## kubelet 핵심 실행 옵션 가이드

`kubelet`은 보통 `/usr/bin/kubelet` 실행 파일 뒤에 옵션을 붙여 사용하거나, `/var/lib/kubelet/config.yaml` 설정 파일을 참조합니다.

| 실무 예제 (Example) | 옵션 (Option) | 용도 설명 |
| --- | --- | --- |
| `kubelet --version` | **--version** | 현재 설치된 kubelet의 버전 정보를 출력 |
| `kubelet --node-ip=192.168.100.1` | **--node-ip** | 노드에 여러 IP가 있을 경우, 클러스터에서 사용할 특정 IP를 명시 |
| `kubelet --container-runtime-endpoint=...` | **--container-runtime-endpoint** | 사용할 CRI(컨테이너 런타임) 소켓 경로 지정 (예: containerd) |
| `kubelet --config=/var/lib/kubelet/config.yaml` | **--config** | 상세 설정을 담은 YAML 형식의 구성 파일 경로 지정 (가장 권장됨) |
| `kubelet --kubeconfig=/etc/kubernetes/kubelet.conf` | **--kubeconfig** | 마스터(API 서버)와 통신하기 위한 인증 및 접속 정보 파일 지정 |
| `kubelet --v=5` | **--v** | 로그 상세 수준 설정 (1~10). 장애 발생 시 숫자 높여 상세 분석 |
| `kubelet --pod-infra-container-image=...` | **--pod-infra-container-image** | 파드 생성 시 기반이 되는 pause 이미지 주소 지정 |
| `kubelet --root-dir=/mnt/kubelet` | **--root-dir** | kubelet의 작업 데이터(볼륨 등)가 저장될 루트 디렉토리 변경 |
| `kubelet --register-node=true` | **--register-node** | API 서버에 이 노드를 자동으로 등록할지 여부 결정 |
| `kubelet --eviction-hard=memory.available<100Mi` | **--eviction-hard** | 노드 자원 부족 시 파드를 강제 종료(Eviction)할 하드웨어 임계치 설정 |

---

## kubelet 관리 및 장애 진단 (강의 강조 포인트)

### 1. 서비스 상태 확인 및 로그 분석

`kubelet`은 백그라운드 서비스로 돌아가므로 `kubectl` 보다는 `systemctl`과 `journalctl`을 더 많이 사용합니다.

* **상태 확인**: `systemctl status kubelet`
* **로그 확인**: `journalctl -u kubelet -f` (실시간 로그 확인 시 필수)

### 2. 설정 방식의 변화 (중요)

최신 쿠버네티스 버전에서는 커맨드라인 옵션(`Flags`)보다는 **`KubeletConfiguration` (YAML 파일)**을 통한 설정을 권장합니다.

* 위치: `/var/lib/kubelet/config.yaml`
* 주요 설정 항목: `cgroupDriver` (cgroupfs vs systemd), `address`, `port` 등

### 3. Cgroup Driver 일치 (자주 발생하는 에러)

Docker나 Containerd의 Cgroup Driver와 kubelet의 설정이 일치하지 않으면 서비스가 시작되지 않습니다.

* **확인 방법**: `grep "cgroup" /var/lib/kubelet/config.yaml`
* **해결**: 컨테이너 런타임과 kubelet 모두 `systemd`로 통일하는 것이 현재 표준입니다.

---

Next Step: kubelet 설정 파일(config.yaml) 상세 구조 및 수정 방법

커맨드라인 옵션 대신 실무에서 주로 사용하는 **`config.yaml`**의 내부 구조와 주요 파라미터를 수정하는 실습을 진행해 볼까요? 혹은 kubelet 로그에서 자주 발생하는 에러 코드 분석을 다뤄볼까요?
