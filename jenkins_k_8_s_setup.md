# Hướng dẫn cài Jenkins kết nối Kubernetes

## 1. Mục tiêu
- Cài Jenkins trong Docker container
- Cài `kubectl` trong container Jenkins
- Kết nối Jenkins với cluster Kubernetes hiện có để deploy ứng dụng (ví dụ Nginx)

## 2. Chuẩn bị
- Ubuntu server với cluster Kubernetes đã setup (`kubeadm init` + cài network plugin như Flannel)
- Node `Ready` trong cluster
- Quyền root hoặc sudo trên host

## 3. Cài Docker trên host
```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```

## 4. Chạy Jenkins container
```bash
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```
- Truy cập Jenkins: `http://<host-ip>:8080`
- Lấy password từ container: `docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword`

## 5. Cài `kubectl` trong container Jenkins
```bash
# Vào container
docker exec -it jenkins bash

# Tải kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
mv kubectl /usr/local/bin/

# Kiểm tra
kubectl version --client
```

## 6. Cấu hình kubeconfig trong Jenkins
- Copy kubeconfig từ host:
```bash
docker cp /home/<user>/.kube/config jenkins:/var/jenkins_home/.kube/config
```
- Trong container Jenkins:
```bash
export KUBECONFIG=/var/jenkins_home/.kube/config
kubectl get nodes
```

## 7. Cài plugin Jenkins cần thiết
- Kubernetes CLI Plugin
- Docker Pipeline
- Git Plugin
- Pipeline Plugin

## 8. Tạo pipeline deploy Nginx
```groovy
pipeline {
    agent any

    environment {
        KUBECONFIG = '/var/jenkins_home/.kube/config'
        DEPLOY_FILE = 'k8s-deployment.yaml'
    }

    stages {
        stage('Deploy Nginx to Kubernetes') {
            steps {
                script {
                    sh """
                    cat <<EOF > $DEPLOY_FILE
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
EOF

                    kubectl apply -f $DEPLOY_FILE --kubeconfig=$KUBECONFIG
                    kubectl get pods -A --kubeconfig=$KUBECONFIG
                    """
                }
            }
        }
    }

    post {
        always { echo "Pipeline finished" }
        success { echo "Nginx deployed thành công!" }
        failure { echo "Có lỗi xảy ra, check log" }
    }
}
```

## 9. Test deploy
- Chạy pipeline
- Kiểm tra pods: `kubectl get pods -A` từ container Jenkins hoặc host
- Truy cập service Nginx: `http://<host-ip>:30080`

## 10. Lưu ý
- Nếu sử dụng Docker in Docker (DinD) trong Jenkins để build image, cần cài thêm Docker client và mount socket.
- Cấu hình `kubeconfig` đúng quyền truy cập với Jenkins.
- Node control-plane có thể cần `taint` bỏ để deploy pods.

