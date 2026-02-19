Terraform(테라폼)을 사용하여 Kubernetes(쿠버네티스) 환경에 Nginx Deployment(엔진엑스 디플로이먼트)를 구축하는 실무 과정을 정리합니다. 이 과정은 테라폼 설치가 완료된 상태를 가정하며, 실제 운영 환경에서 사용하는 디렉토리 구조와 베스트 프랙티스를 반영합니다.

---

## 테라폼을 이용한 Nginx Deployment 구축

### 1. 작업 디렉토리 구성

실무에서는 관리의 편의성과 모듈화를 위해 프로젝트별로 디렉토리를 분리합니다. 먼저 터미널에서 작업 공간을 생성합니다.

```bash
mkdir -p terraform-k8s-nginx
cd terraform-k8s-nginx
touch provider.tf main.tf variables.tf outputs.tf

```

---

### 2. Provider 설정 (`provider.tf`)

테라폼이 어떤 대상(AWS, Azure, Kubernetes 등)과 통신할지 정의합니다. 여기서는 로컬 또는 원격 쿠버네티스 클러스터에 접속하기 위한 설정을 작성합니다.

```hcl
# Kubernetes Provider 설정
provider "kubernetes" {
  # 로컬의 kubeconfig 파일을 참조하여 클러스터에 접속합니다.
  config_path = "~/.kube/config"
}

```

---

### 3. 변수 정의 (`variables.tf`)

코드의 재사용성을 높이기 위해 변경 가능성이 있는 값들을 변수로 분리합니다.

```hcl
variable "nginx_labels" {
  description = "Nginx 리소스에 적용할 라벨"
  type        = map(string)
  default = {
    App  = "ScalableNginx"
    Tier = "Frontend"
  }
}

variable "replica_count" {
  description = "생성할 파드(Pod)의 개수"
  type        = number
  default     = 3
}

```

---

### 4. 메인 리소스 정의 (`main.tf`)

실제 Nginx Deployment와 이를 외부로 노출할 Service(서비스)를 정의합니다.

```hcl
# Nginx Deployment 정의
resource "kubernetes_deployment" "nginx_deploy" {
  metadata {
    name = "nginx-deployment"
    labels = var.nginx_labels
  }

  spec {
    replicas = var.replica_count # 변수 처리된 복제본 수

    selector {
      match_labels = var.nginx_labels
    }

    template {
      metadata {
        labels = var.nginx_labels
      }

      spec {
        container {
          image = "nginx:1.21.6" # Nginx 이미지 버전 명시
          name  = "nginx-server"

          port {
            container_port = 80
          }

          resources {
            limits = {
              cpu    = "0.5"
              memory = "512Mi"
            }
            requests = {
              cpu    = "250m"
              memory = "50Mi"
            }
          }
        }
      }
    }
  }
}

# Nginx Service 정의 (외부 노출용)
resource "kubernetes_service" "nginx_svc" {
  metadata {
    name = "nginx-service"
  }
  spec {
    selector = var.nginx_labels
    port {
      port        = 80
      target_port = 80
    }
    type = "NodePort" # 클라우드 환경인 경우 LoadBalancer(로드밸런서) 할당
  }
}

```

---

### 5. 결과 출력 설정 (`outputs.tf`)

배포 완료 후 확인이 필요한 정보를 화면에 출력합니다.

```hcl
# 배포된 서비스의 접속 정보 출력
output "service_access_url" {
  description = "배포된 Nginx 서비스의 접속 URL"
  # VM의 IP는 고정되어 있다고 가정하거나, 환경에 맞춰 출력합니다.
  value       = "http://[VM_IP_ADDRESS]:${kubernetes_service.nginx_svc.spec.0.port.0.node_port}"
}
```

---

### 6. 인프라 배포 및 관리 명령어

파일 작성이 완료되면 다음 순서로 명령어를 실행하여 배포를 진행합니다.

1. **초기화 (Initialize):** 프로바이더 플러그인을 다운로드합니다.
```bash
terraform init

```


2. **계획 확인 (Plan):** 생성될 리소스를 미리 확인합니다.
```bash
terraform plan

```


3. **적용 (Apply):** 실제 클러스터에 리소스를 생성합니다.
```bash
terraform apply -auto-approve

```



---

> **[주석 및 실무 팁]**
> * **LoadBalancer(로드밸런서):** 클라우드 공급자(AWS, GCP 등) 환경에서 이 설정을 사용하면 실제 외부 IP가 할당되며, 이 과정에서 **클라우드 과금이 발생**하므로 주의가 필요합니다. 로컬 테스트(Minikube 등) 환경이라면 `NodePort` 타입을 권장합니다.
> * **State File(상태 파일):** 실무에서는 `terraform.tfstate` 파일을 로컬이 아닌 S3나 Terraform Cloud 같은 원격 저장소(Backend)에 저장하여 팀원 간의 충돌을 방지합니다.
> 
> 

---

**Next Step: 테라폼 백엔드 설정을 통한 상태 파일(State File) 공유 및 협업 환경 구축**

안녕하십니까. 요청하신 설정에 맞춰 실무 환경에서 바로 활용 가능한 구조로 교재 본문을 구성했습니다. 모든 전문 용어는 전체 명칭과 한글 설명을 병기하였으며, 과금 주의 사항을 포함했습니다. 추가로 수정이 필요한 부분이 있다면 말씀해 주세요.
