
이 안내서는 로컬 PC(Windows/macOS)에 **Terraform (테라폼)**을 설치하고, 사용자가 보유한 **Ubuntu VM (우분투 가상 머신)** 내의 **Kubernetes (쿠버네티스, K8s)** 클러스터와 연결하기 위한 사전 준비 과정을 다룹니다.

---

## 1. 로컬 PC: Terraform (테라폼) 설치

테라폼은 인프라를 코드로 관리하는 도구입니다. PC 운영체제에 맞춰 설치를 진행합니다.

### 가. Windows 사용자 (Chocolatey 사용)

1. **관리자 권한**으로 PowerShell 실행
2. `choco install terraform` 입력 후 설치
3. `terraform -version`으로 설치 확인

### 나. macOS 사용자 (Homebrew 사용)

1. 터미널 실행
2. `brew tap hashicorp/tap` 및 `brew install hashicorp/tap/terraform` 입력
3. `terraform -version`으로 설치 확인

---

## 2. Ubuntu VM: Kubernetes (쿠버네티스) 접속 정보 확인

테라폼이 VM 내부의 K8s 클러스터에 명령을 내리려면 **kubeconfig (쿠브설정)** 파일이 필요합니다.

1. **Ubuntu VM 접속**: SSH를 통해 VM에 로그인합니다.
2. **설정 파일 확인**: K8s 관리자 설정 파일 내용을 확인합니다.
```bash
# 설정 파일 내용 출력
cat ~/.kube/config

```


3. **외부 접속 허용 확인**:
* `clusters.cluster.server` 항목의 IP가 `127.0.0.1`로 되어 있다면 외부(로컬 PC)에서 접속할 수 없습니다.
* VM의 **Public IP (공인 IP)** 또는 **Private IP (사설 IP)**로 변경되어 있는지 확인하세요.



---

## 3. 로컬 PC와 VM 간의 연결 설정 (kubeconfig 복사)

테라폼이 실행될 로컬 PC에 VM의 K8s 인증 정보를 가져와야 합니다.

1. **로컬 PC에서 디렉토리 생성**:
* Windows: `C:\Users\[사용자명]\.kube`
* macOS: `~/.kube`


2. **파일 복사 (SCP 활용)**: 로컬 터미널에서 아래 명령어를 실행하여 VM의 설정 파일을 로컬로 가져옵니다.
```bash
# VM의 config 파일을 로컬 PC의 .kube 폴더로 복사
scp ubuntu@[VM_IP]:~/.kube/config ~/.kube/config_ubuntu_vm

```


3. **환경 변수 설정**: 테라폼이 이 파일을 참조하도록 설정합니다.
* **Windows (PowerShell)**: `$env:KUBECONFIG="C:\Users\[사용자명]\.kube\config_ubuntu_vm"`
* **macOS**: `export KUBECONFIG=~/.kube/config_ubuntu_vm`



---

## 4. Terraform 프로젝트 초기화 및 Provider 설정

이제 테라폼 코드에서 Kubernetes를 제어할 수 있도록 **Provider (공급자)**를 설정합니다.

1. **작업 폴더 생성**: `mkdir k8s-terraform-lab && cd k8s-terraform-lab`
2. **provider.tf 파일 생성**: 아래 내용을 입력하여 테라폼과 K8s를 연결합니다.

```hcl
# Kubernetes Provider 설정
provider "kubernetes" {
  config_path = "~/.kube/config_ubuntu_vm" # 복사해온 설정 파일 경로
}

# 테스트용 네임스페이스 리소스 정의
resource "kubernetes_namespace" "example" {
  metadata {
    name = "terraform-test-space"
  }
}

```

3. **초기화 실행**:
```bash
# K8s용 플러그인 다운로드
terraform init

```



---

## 5. 연결 테스트 및 보안 주의사항

1. **연결 확인**: `terraform plan` 명령어를 입력하여 에러 없이 실행 계획이 출력되는지 확인합니다.
2. **네트워크 보안**: VM의 방화벽에서 K8s API 서버 포트(기본 **6443**)가 로컬 PC의 IP로부터 접근 허용되어 있어야 합니다. (**보안 그룹 설정 확인**)
3. **인증서 주의**: 만약 VM의 IP가 바뀌었다면 `config` 파일 내의 `certificate-authority-data` 관련 에러가 발생할 수 있습니다. 이때는 `insecure = true` 옵션을 임시로 사용할 수 있으나 보안상 권장되지 않습니다.

---

### 주석 (Annotations)

* **K8s (Kubernetes, 쿠버네티스)**: 컨테이너화된 애플리케이션의 배포, 확장 등을 자동화하는 오픈소스 플랫폼입니다.
* **KUBECONFIG (쿠브설정)**: 쿠버네티스 클러스터에 접속하기 위한 인증 정보와 서버 주소가 담긴 설정 파일입니다.
* **Provider (공급자)**: 테라폼이 특정 플랫폼(AWS, Kubernetes 등)과 통신할 수 있게 해주는 라이브러리입니다.
* **SCP (Secure Copy Protocol, 보안 복사 프로토콜)**: 네트워크를 통해 파일을 안전하게 전송하는 프로토콜입니다.
* **Namespace (네임스페이스)**: 단일 쿠버네티스 클러스터 내에서 리소스를 격리하는 논리적 그룹입니다.

Next Step: 테라폼을 이용한 K8s Deployment(디플로이먼트) 및 Service(서비스) 생성 실습
