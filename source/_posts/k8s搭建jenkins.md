---
title: k8s搭建jenkins
date: 2026-06-29 15:58:47
tags: [K8s、Jenkins]
---

#### 创建命名空间

```shell
kubectl create ns jenkins
kubectl create ns dev
```

#### 创建 Jenkins 的 ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-sa
  namespace: jenkins
```

```shell
kubectl apply -f jenkins-sa.yaml
```

#### 给 Jenkins 授权操作 dev 环境

jenkins-role.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: jenkins-deploy-role
  namespace: dev
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "patch", "update"]

  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]

  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "list", "watch"]
```

jenkins-rolebinding.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-deploy-binding
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: jenkins-sa
    namespace: jenkins
roleRef:
  kind: Role
  name: jenkins-deploy-role
  apiGroup: rbac.authorization.k8s.io
```

```shell
# jenkins 命名空间里的 jenkins-sa 可以操作 dev 命名空间里的 Deployment
kubectl apply -f jenkins-role.yaml
kubectl apply -f jenkins-rolebinding.yaml
```

#### 部署 Jenkins 到 K8s

jenkins-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins-sa
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-home
          emptyDir: {}
```

jenkins-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  type: NodePort
  selector:
    app: jenkins
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080
```

应用

```shell
kubectl apply -f jenkins-deployment.yaml
kubectl apply -f jenkins-service.yaml
```

访问 http://节点IP:30080

注：生产环境不要用 emptyDir，要换成 PVC，否则 Jenkins 重启数据会丢。

#### 业务项目 Deployment

假设你的服务叫 backend。
backend-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: registry.cn-hangzhou.aliyuncs.com/demo/backend:init
          ports:
            - containerPort: 9999
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 9999
            initialDelaySeconds: 20
            periodSeconds: 5
```

backend-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: dev
spec:
  selector:
    app: backend
  ports:
    - port: 9999
      targetPort: 9999
```

应用

```shell
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml
```

Jenkinsfile 示例
这里先用最容易理解的版本：Jenkins 负责打包，然后更新 K8s。

```yaml
pipeline {
agent any

    environment {
        APP_NAME = 'backend'
        NAMESPACE = 'dev'
        IMAGE_REPO = 'registry.cn-hangzhou.aliyuncs.com/demo/backend'
        IMAGE_TAG = "${BUILD_NUMBER}"
        IMAGE = "${IMAGE_REPO}:${IMAGE_TAG}"
    }

    stages {
        stage('拉取代码') {
            steps {
                checkout scm
            }
        }

        stage('Maven 打包') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('构建并推送镜像') {
            steps {
                sh '''
                echo "这里推荐用 Kaniko / BuildKit 构建镜像"
                echo "镜像版本: ${IMAGE}"
                '''
            }
        }

        stage('部署到 K8s') {
            steps {
                sh '''
                kubectl set image deployment/${APP_NAME} \
                  ${APP_NAME}=${IMAGE} \
                  -n ${NAMESPACE}

                kubectl rollout status deployment/${APP_NAME} -n ${NAMESPACE}
                '''
            }
        }
    }

    post {
        failure {
            sh '''
            echo "发布失败，可以执行回滚："
            echo "kubectl rollout undo deployment/${APP_NAME} -n ${NAMESPACE}"
            '''
        }
    }

}
```

#### 权限验证

进入 Jenkins Pod：

```shell
kubectl exec -it -n jenkins deploy/jenkins -- bash
```

测试权限

```shell
kubectl get pods -n dev
kubectl get deployment -n dev
```

测试不能操作其他 namespace

```shell
kubectl get pods -n kube-system
```

如果权限控制正常，这个应该会报Forbidden

#### 整体理解

```shell
Jenkins Pod
      ↓
serviceAccountName: jenkins-sa
      ↓
jenkins-sa
      ↓
RoleBinding
      ↓
jenkins-deploy-role
      ↓
允许更新 dev namespace 的 Deployment
```
