---

## 1. 스토리지 준비: NFS 서버 설정

쿠버네티스 노드들이 공용 저장소를 사용하기 위해 ip 10번을 NFS 서버로 사용했을 때의 Script

### NFS 설치 및 디렉토리 생성

```bash
# VM 10번(스토리지 서버)에서 실행
sudo apt-get update
sudo apt-get install -y nfs-kernel-server # NFS 서버 패키지 설치

# 공유 디렉토리 생성 및 권한 부여
sudo mkdir -p /mnt/nfs_share
sudo chown nobody:nogroup /mnt/nfs_share
sudo chmod 777 /mnt/nfs_share

# NFS export 설정 (사내 네트워크 대역 허용)
echo "/mnt/nfs_share *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
sudo exportfs -a
sudo systemctl restart nfs-kernel-server

```

---

## 2. 쿠버네티스 PV 및 PVC 구성 (YAML)

전체 5GB의 PV를 생성하고, 각 서비스가 요청한 용량만큼 PVC를 통해 할당받습니다.

### `storage-resource.yaml`

```yaml
# 1. 영구 볼륨 (PV) 정의 - 전체 5GB
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-5gb
spec:
  capacity:
    storage: 5Gi         # 전체 용량 5GB 설정
  accessModes:
    - ReadWriteMany      # 여러 노드에서 동시 읽기/쓰기 허용
  nfs:
    server: 192.168.109.10 # 환경에 따라 IP 바꾸기
    path: /mnt/nfs_share # NFS 공유 경로
---
# 2. 프론트엔드용 PVC (1GB)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: frontend-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi       # 1GB 할당 요청
---
# 3. 백엔드용 PVC (1GB)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: backend-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi       # 1GB 할당 요청

```

---

## 3. 백엔드 구성: Flask 및 PVC 연동

백엔드는 `/data` 경로에 마운트된 PV 내 파일 리스트를 JSON으로 반환합니다.

### `backend-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: flask
        image: python:3.9-slim
        command: ["python", "-c"]
        # 파일 리스트를 반환하는 간단한 Flask 코드
        args: 
        - |
          from flask import Flask, jsonify
          import os
          app = Flask(__name__)
          @app.route('/api/files')
          def get_files():
              files = os.listdir('/data') # PV 마운트 경로 읽기
              return jsonify(files)
          app.run(host='0.0.0.0', port=5000)
        ports:
        - containerPort: 5000
        volumeMounts:
        - name: data-storage
          mountPath: /data # 컨테이너 내부 마운트 경로
      volumes:
      - name: data-storage
        persistentVolumeClaim:
          claimName: backend-pvc # 백엔드용 1GB PVC 연결
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5000

```

---

## 4. 프론트엔드 구성: Nginx 및 백엔드 연결 버튼

프론트엔드는 PVC의 파일 목록을 보여주고 백엔드로 이동하는 버튼을 포함합니다.

### `frontend-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        volumeMounts:
        - name: html-storage
          mountPath: /usr/share/nginx/html # Nginx 기본 웹 경로에 마운트
      volumes:
      - name: html-storage
        persistentVolumeClaim:
          claimName: frontend-pvc # 프론트엔드용 1GB PVC 연결
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: NodePort # 외부 접속을 위해 NodePort 설정
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30001 # 브라우저 접속 포트

```

---

## 5. 실행 및 테스트 방법

### 1) 리소스 배포

```bash
kubectl apply -f storage-resource.yaml
kubectl apply -f backend-deployment.yaml
kubectl apply -f frontend-deployment.yaml

```

### 2) 테스트용 파일 생성

NFS 서버(ip 10번)에서 파일을 생성하여 PV가 정상 작동하는지 확인합니다.

```bash
echo "Hello from PV" > /mnt/nfs_share/testfile.txt

```

### 3) 브라우저 접속 테스트

1. **프론트엔드 접속:** 브라우저에서 `http://[노드IP]:30001` 접속.
2. **파일 리스트 확인:** 화면에 `testfile.txt`가 보이는지 확인. (Nginx 설정에 따라 `autoindex on;` 처리가 필요할 수 있습니다.)
3. **백엔드 이동:** 하단의 버튼을 클릭하여 `http://[노드IP]:30001/api/files` (혹은 설정한 경로)로 이동하여 JSON 데이터가 나오는지 확인합니다.

---

**Next Step:** Nginx Ingress를 활용한 도메인 기반 라우팅 설정

위 절차는 VM 50번을 스토리지 전용 서버로 활용하고, K8s 클러스터 내에서 용량을 나누어 사용하는 인프라의 기본 모델입니다.

Next Step: **Ingress Routing**
