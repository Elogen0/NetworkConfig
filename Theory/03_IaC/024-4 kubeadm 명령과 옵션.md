쿠버네티스 클러스터를 구축하고 관리하는 핵심 도구인 `kubeadm`의 주요 명령어와 옵션을 정리해 드립니다. 

---

## kubeadm 핵심 명령어 및 옵션 가이드

| 실무 예제 (Example) | 명령어 (Command) | 주요 옵션 (Options) | 용도 설명 |
| --- | --- | --- | --- |
| `kubeadm init --pod-network-cidr=10.244.0.0/16` | **init** | `--pod-network-cidr`, `--apiserver-advertise-address` | **마스터 노드(Control Plane)**를 초기화하고 클러스터를 시작 |
| `kubeadm join 192.168.0.1:6443 --token...` | **join** | `--token`, `--discovery-token-ca-cert-hash` | **워커 노드**를 기존 클러스터에 합류시킴 |
| `kubeadm reset -f` | **reset** | `-f` (force), `--cri-socket` | 해당 노드의 모든 쿠버네티스 설정을 삭제하고 **초기화** |
| `kubeadm token create --print-join-command` | **token** | `create`, `list`, `delete`, `--print-join-command` | 노드 합류에 필요한 **인증 토큰** 관리 및 조인 명령어 재생성 |
| `kubeadm upgrade plan` | **upgrade** | `plan`, `apply`, `node` | 클러스터를 다음 버전으로 **업그레이드**하기 위한 실행 및 계획 확인 |
| `kubeadm config images pull` | **config** | `images pull`, `print init-defaults` | 설치에 필요한 **컨테이너 이미지**를 미리 다운로드하거나 설정 파일 생성 |
| `kubeadm certs check-expiration` | **certs** | `check-expiration`, `renew` | 클러스터 내부 통신용 **인증서** 만료 확인 및 갱신 |
| `kubeadm version -o json` | **version** | `-o json`, `-o short` | 설치된 kubeadm의 버전 정보를 확인 |

---

## 단계별 실행 흐름 및 주의사항 (강의 강조 포인트)

### 1. 클러스터 생성 단계 (Master Node)

* **`init` 수행 시**: 반드시 CNI(네트워크 플러그인)가 요구하는 `--pod-network-cidr` 대역을 확인해야 합니다. (예: Flannel은 `10.244.0.0/16`, Calico는 `192.168.0.0/16` 권장)
* **메모리/CPU**: 최소 2GB RAM과 2 Core CPU가 필요하며, 부족할 경우 `preflight` 체크에서 에러가 발생합니다.

### 2. 노드 확장 단계 (Worker Node)

* **조인 명령어 분실 시**: 마스터 노드에서 `kubeadm token create --print-join-command`를 실행하면 해시값까지 포함된 전체 명령어가 출력되어 편리합니다.
* **토큰 유효기간**: 기본 토큰은 생성 후 **24시간** 뒤 만료되므로, 새로 조인할 때는 토큰을 새로 생성하는 것이 좋습니다.

### 3. 문제 해결 단계 (Troubleshooting)

* **재설치 시**: 환경이 꼬였다면 `kubeadm reset -f`를 실행한 후, 반드시 `/etc/cni/net.d`, `$HOME/.kube`, `/var/lib/etcd` 등을 수동으로 삭제해야 깨끗한 재설치가 가능합니다.

---

