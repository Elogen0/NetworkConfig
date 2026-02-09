
### 인벤토리 파일
각 노드에대한 정보를 담은 인벤토리 파일을 생성합니다. 로컬환경은 ansible_connection=local을 추가해야 합니다.<br>
**host.ini**
```
[k8s_master]
192.168.109.251  ansible_connection=local

[k8s_worker]
192.168.109.1
192.168.109.2

[all:vars]
ansible_user=master
```

### 쿠버네티스 설치를 위한 Ansible Playbook
복제된 VM의 식별자 충돌 방지와 컨테이너 런타임 최적화를 포함한 파일입니다.

**k8s_setup.yaml**
```yaml
---
- name: Kubernetes v1.30 환경 최적화 및 설치 (최종본)
  hosts: all
  become: yes
  vars:
    internal_network: "192.168.109.0/24"  # 실습 환경 대역에 맞춰 수정

  tasks:
    # [방화벽] 노드 간 통신을 위해 특정 대역 및 포트 허용
    - name: 방화벽 설정 - 내부 대역 및 필수 포트 개방
      ufw:
        rule: allow
        port: "{{ item.port | default(omit) }}"
        from_ip: "{{ item.from | default(omit) }}"
        proto: tcp
      loop:
        - { from: "{{ internal_network }}" }      # 내부 노드 간 전 포트 신뢰
        - { port: '22' }                          # SSH 관리용
        - { port: '6443' }                        # API Server 전용
        - { port: '10250' }                       # Kubelet API
        - { port: '30000:32767' }                 # 외부 서비스 노출용 NodePort 대역

    - name: 방화벽 활성화
      ufw:
        state: enabled
        policy: deny

    # [OS 최적화] K8s 구동을 위한 커널 및 메모리 설정
    - name: 필수 도구 및 스왑 비활성화
      apt:
        name: [ca-certificates, curl, gnupg, ufw]
        update_cache: yes

    - name: Swap 메모리 비활성화 (K8s 성능 및 안정성 필수 요구사항)
      shell: |
        swapoff -a
        sed -i '/swap/s/^/#/' /etc/fstab

    - name: 커널 모듈 및 파라미터 적용 (L2 브리지 트래픽 가시성 확보)
      copy:
        dest: /etc/modules-load.d/k8s.conf
        content: "overlay\nbr_netfilter"

    - name: 모듈 즉시 로드 및 sysctl 설정
      shell: |
        modprobe overlay && modprobe br_netfilter
        sysctl --system

    # [런타임] Containerd 설치 및 Cgroup 드라이버 설정
    - name: Containerd 설치 및 설정 (SystemdCgroup 활성화 핵심)
      shell: |
        apt-get install -y containerd
        mkdir -p /etc/containerd
        containerd config default > /etc/containerd/config.toml
        # Kubelet의 Cgroup 관리 방식과 일치시키기 위해 true로 변경
        sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
      notify: restart containerd

    # [K8s 패키지] 저장소 정비 및 v1.30 설치
    - name: GPG 키 및 저장소 청소 (기존 인증 에러 방지용)
      shell: |
        rm -f /etc/apt/keyrings/kubernetes-apt-keyring.gpg
        rm -f /etc/apt/sources.list.d/kubernetes.list
        apt-get clean && rm -rf /var/lib/apt/lists/*

    - name: K8s v1.30 GPG 키 다운로드 및 변환
      shell: |
        mkdir -p /etc/apt/keyrings
        curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
      args:
        creates: /etc/apt/keyrings/kubernetes-apt-keyring.gpg

    - name: K8s 소스 리스트 추가 및 패키지 설치
      apt_repository:
        repo: "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /"
        filename: kubernetes

    - name: 기존 kube 패키지 hold 해제
      shell: |
        apt-mark unhold kubelet kubeadm kubectl
      ignore_errors: yes

    - name: K8s도구 설치 및 버전 업데이트 고정 (운영 중 버전 꼬임 방지)
      apt:
        name:
          - kubelet=1.30.*
          - kubeadm=1.30.*
          - kubectl=1.30.*
        state: present
        update_cache: yes

    - name: 패키지 홀드 설정
      dpkg_selections:
        name: "{{ item }}"
        selection: hold
      loop: [kubelet, kubeadm, kubectl]


    # [권한 설정] 마스터 노드 관리자 설정
    - name: kubeconfig 디렉토리 생성 및 권한 설정
      file:
        path: "{{ ansible_facts['env']['HOME'] }}/.kube"
        state: directory
        mode: '0755'

  handlers:
    - name: restart containerd
      service:
        name: containerd
        state: restarted
```

마스터 PC에서 아래 명령어를 실행합니다.
```
export ANSIBLE_HOST_KEY_CHECKING=False
ansible-playbook -i hosts.ini k8s_setup.yaml -K
```

---

### 마스터 노드 세팅
**k8s_master_init.yaml**
```yaml
---
- name: Initialize Kubernetes Master Node
  hosts: k8s_master
  become: yes

  tasks:
    - name: Check if Kubernetes master is already initialized
      ansible.builtin.stat:
        path: /etc/kubernetes/admin.conf
      register: k8s_init_check

    - name: Initialize the Kubernetes Cluster (kubeadm init with config)
      ansible.builtin.command: >
        kubeadm init --config kubeadm-init.yaml
      when: not k8s_init_check.stat.exists

    - name: Create .kube directory for cluster config
      ansible.builtin.file:
        path: /home/{{ ansible_user }}/.kube
        state: directory
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: '0755'

    - name: Copy admin.conf to user's .kube directory
      ansible.builtin.copy:
        src: /etc/kubernetes/admin.conf
        dest: /home/{{ ansible_user }}/.kube/config
        remote_src: yes
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: '0600'

    - name: Wait for Kubernetes API server
      ansible.builtin.command: kubectl get nodes
      environment:
        KUBECONFIG: /etc/kubernetes/admin.conf
      register: api_check
      retries: 10
      delay: 10
      until: api_check.rc == 0

    - name: Install Flannel CNI Network Plugin
      ansible.builtin.command: >
        kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
      environment:
        KUBECONFIG: /etc/kubernetes/admin.conf

    - name: Generate join command for worker nodes
      ansible.builtin.command: >
        kubeadm token create --print-join-command
      register: join_command_result

    - name: Display join command
      ansible.builtin.debug:
        msg: "Worker Join Command: {{ join_command_result.stdout }}"
      delegate_to: localhost
```

마스터 노드에서 `ansible-playbook -i hosts.ini k8s_master_init.yaml -K` 명령어로 실행시킵니다.

---

## Kubernetes Control Plane 구축
클러스터를 처음 구축할 때(Provisioning), 마스터 노드(Control Plane)를 어떻게 구성할 것인지에 대한 선택은 운영 효율성과 유지보수 측면에서 매우 중요합니다. 
 **`kubeadm` 명령어를 사용하되, 설정값은 YAML 파일(Configuration File)로 관리하는 방식**이 가장 권장됩니다.
 
#### 1. 명령형 설치 (`kubeadm init` 옵션 사용)
명령어 뒤에 다양한 플래그(Flag)를 붙여 마스터 노드를 초기화하는 방식입니다.
* **예시:** `kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.1.10 ...`
* **장점:** 별도의 파일 생성 없이 한 줄로 빠르게 클러스터를 띄울 수 있어 1회성 테스트에 적합합니다.
* **단점:** 설정 항목이 많아질수록 명령어가 길어지고 복잡해집니다. 나중에 똑같은 설정을 재현하거나 팀원과 공유하기 어렵습니다.

#### 2.선언형 설치 (`kubeadm init --config` 파일 사용)
모든 설치 옵션을 YAML 형식의 설정 파일에 미리 작성하고 이를 참조하여 설치하는 방식입니다.
* **예시:** `kubeadm init --config=kubeadm-config.yaml`
* **장점:**
* **문서화:** 어떤 옵션으로 마스터가 구성되었는지 파일 하나로 명확히 알 수 있습니다.
* **정밀한 제어:** CLI 플래그로 제공되지 않는 세부적인 API Server, Scheduler, Controller Manager의 파라미터를 YAML 내에서 상세히 조정할 수 있습니다.
* **재현성:** 클러스터를 재구축하거나 확장할 때 동일한 환경을 보장합니다.

### 선언형 방식 설치 방식 프로세스
kubeadm config 기본 설정 파일 추출
```
kubeadm config print init-defaults > kubeadm-config.yaml
```
생성된 kubeadm-config.yaml파일을 열어 환경에 맞게 수정합니다.
에러가 그대로 쓰면 에러가 생성되는데, podSubnet을 추가해야 합니다.

**kubeadm-config.yaml**
```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.109.251
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: KubeMaster
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io
kind: ClusterConfiguration
kubernetesVersion: 1.30.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16
scheduler: {}
```

### kube-adm init(마스터에서 실행)
```
kubeadm init --config=kubeadm-config.yaml
```
이게 실행이 되면 마스터에서의 클러스터 환경을 만든다. 쿠버네티스 클러스터를 “처음 만들 때” 마스터(컨트롤 플레인)를 초기화하는 명령어
- 아래 컨트롤 플레인을 Pod 형태로 띄움
  - API Server
  - Scheduler
  - Controller Manager
  - etcd
- 인증서 생성
  - CA (ca.crt)
  - API Server 인증서
  - kubelet 인증서
  - 📁 위치: `/etc/kubernetes/pki/`
- kubeconfig 생성
  - admin.conf
  - controller-manager.conf
  - scheduler.conf
  - kubelet.conf
  - 📁 위치: `/etc/kubernetes/`
- 클러스터 “정체성” 생성
  - 클러스터 이름
  - 서비스 CIDR
  - Pod 네트워크 CIDR
  - Kubernetes 버전
  - “이 클러스터에 들어와도 된다”는 기준 생성
→ CA + API Server Endpoint

---

## 워커 노드 Join
세팅된 마스터에 워커 노드가 알아서 붙을수 있도록 Join 세팅을 해줍니다.
### kubeadm 토큰 생성
```
kubeadm token create --print-join-command --certificate-key $(kubeadm init phase upload-certs --upload-certs | tail -1)
```
만들어진 클러스터에 들어갈 열쇠를 만든다
“새 노드가 이 클러스터에 들어와도 된다”는 입장권(Token)을 만드는 명령어

**생성된 adm-token 내용**
```
kubeadm join 192.168.109.251:6443 --token cv12sz.s4bcbw1sazqh2vhe --discovery-token-ca-cert-hash sha256:c042559fa7680cc9492a6343123f35ecd3ea1e6f00325c4c4588abedc4d0d263
```

### kubeadm join파일 작성
생성된 토큰을 통해 워커들이 마스터로 join할수 있게 yaml 파일을 작성한다.<br>
token 과 caCertHashes를 위에서 생성한 토큰 값으로 변경한다.<br>
apiServerEndpint에는 마스터노드의 ip를 입력한다.<br>

**kubeadm-join.yaml**
```
apiVersion: kubeadm.k8s.io/v1beta3
kind: JoinConfiguration
discovery:
  bootstrapToken:
    apiServerEndpoint: 192.168.109.251:6443
    token: cv12sz.s4bcbw1sazqh2vhe
    caCertHashes:
      - sha256:c042559fa7680cc9492a6343123f35ecd3ea1e6f00325c4c4588abedc4d0d263
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
```

### 해당 파일을 사용하여 워커노드 join 
**k8s_worker_join.yaml**
```
---
- name: Join Worker Nodes to Cluster
  hosts: k8s_worker
  become: yes

  tasks:
    - name: Copy kubeadm join config to worker nodes
      ansible.builtin.copy:
        src: kubeadm-join.yaml
        dest: /root/kubeadm-join.yaml
        mode: '0600'

    - name: Join worker node to Kubernetes cluster using config
      ansible.builtin.command: >
        kubeadm join --config /root/kubeadm-join.yaml
      register: join_output
      changed_when: join_output.rc == 0
```

마스터 노드에서 `ansible-playbook -i hosts.ini k8s_worker_join.yaml -K` 명령어로 실행한다

---

# 설치 실패시 싹 밀어버리기

### k8s_reset.yaml
```yaml
---
- name: Kubernetes 클러스터 환경 강제 초기화
  hosts: all
  become: yes
  tasks:
    - name: kubeconfig 존재 여부 확인
      stat:
        path: /etc/kubernetes/admin.conf
      register: kube_config

    - name: 클러스터 정보 강제 삭제 (멈춤 방지 로직 적용)
      shell: |
        # 1. 서비스 중지 및 관련 프로세스 강제 종료
        systemctl stop kubelet || true
        killall -9 kubelet kube-proxy 2>/dev/null || true

        # 2. 런타임 내 실행 중인 모든 컨테이너 중지
        crictl ps -q | xargs -r crictl stop || true

        # 3. 마운트된 볼륨 해제 (Lazy unmount로 프로세스 점유 무관하게 해제)
        df | grep /var/lib/kubelet | awk '{print $6}' | xargs -r umount -l || true

        # 4. kubeadm 초기화 실행
        kubeadm reset -f

        # 5. CNI 설정 및 잔여 데이터 완전 삭제
        rm -rf /etc/cni/net.d /var/lib/kubelet/* /etc/kubernetes/*
        ip link delete cni0 || true
        ip link delete flannel.1 || true

        # 6. iptables 규칙 초기화 (네트워크 꼬임 방지 핵심)
        iptables -F && iptables -t nat -F && iptables -X
      when: kube_config.stat.exists
      async: 120  # 대규모 볼륨 해제 시 멈춤 방지를 위해 비동기 처리
      poll: 5
```
