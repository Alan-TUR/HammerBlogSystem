# HammerBlogSystem 基于k8s集群的锤子博客高可用架构

#### 介绍
本项目是一个以 Kubernetes（k8s）为核心的分布式高可用博客系统，旨在通过微服务架构和容器化技术实现弹性扩展、故障自愈和性能优化。

#### 软件架构
- 整体架构：采用Spring Cloud微服务框架，结合k8s集群实现服务治理、负载均衡和自动化运维。 
  - 服务拆分：博客核心功能拆分为用户服务、文章服务、评论服务、文件存储服务等独立模块，通过Nacos注册中心实现服务发现与动态配置。
  - 流量治理：使用Sentinel实现接口级限流、熔断和降级，保障核心业务高可用。
- 存储层：
  - MySQL集群：基于主从复制+读写分离（使用ShardingSphere分库分表）提升读写性能，通过分布式事务AT模式（Seata框架）保证数据一致性。
  - Redis集群：采用双写模式+哨兵机制实现缓存高可用，缓解数据库压力并提升热点数据访问速度。
- 消息中间件：RocketMQ/Kafka处理异步任务（如日志采集、文章更新通知），通过消息持久化和事务消息确保数据可靠性。
- 容器化部署：所有服务打包为Docker镜像，由k8s集群统一调度和管理，结合HPA（水平自动扩缩容）应对流量峰值。

### 核心技术实现
- Java与JVM优化：
  - 基于JVM内存模型（堆栈划分、GC算法）进行参数调优，减少Full GC频率；使用jstack、Arthas排查线程死锁和内存泄漏问题。
  - 并发编程：通过ReentrantLock、AQS实现分布式锁（Redis+Lua脚本），避免超卖问题；线程池优化任务处理效率。
- 微服务与分布式技术：
  - 前端基于 Vue 脚手架工程创建
  - Spring Cloud Alibaba生态：Nacos（服务注册与配置中心）、Sentinel（流量控制）、Seata（分布式事务）。
  - 服务间通信：OpenFeign+Ribbon实现声明式HTTP调用，结合Hystrix隔离故障服务。
- 数据库与缓存：
  - MySQL优化：通过Explain分析慢SQL，建立联合索引、使用覆盖索引减少回表；利用Binlog实现数据同步。
  - Redis应用：采用ZSet实现文章排行榜，Hash结构存储用户会话信息，布隆过滤器防止缓存穿透。
- 运维与监控：
  - 使用Prometheus+Grafana监控容器资源（CPU/内存）和JVM指标；通过ELK（Elasticsearch+Logstash+Kibana）收集日志。
  - Jenkins Pipeline实现CI/CD自动化部署，结合k8s Rolling Update策略实现零停机发布。

#### 安装教程
#### 1. 环境准备
##### 1.1 基础依赖安装
```bash
# 安装 Docker（所有节点）
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce docker-ce-cli containerd.io
sudo systemctl start docker && sudo systemctl enable docker

# 安装 kubectl、kubeadm、kubelet（Master 和 Worker 节点）
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
sudo yum install -y kubelet kubeadm kubectl
sudo systemctl enable kubelet

# 初始化 k8s 集群（仅 Master 节点）
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 安装网络插件（如 Calico）
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# 加入 Worker 节点（在 Worker 节点执行 kubeadm join 命令，根据初始化输出）
```

##### 1.2 工具安装
```bash
# 安装 Helm（包管理工具）
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

# 安装 Git、Maven、JDK 17
sudo yum install -y git maven java-17-openjdk-devel
```

#### 2. 中间件与数据库部署
##### 2.1 MySQL 集群（主从复制 + 读写分离）
```bash
# 使用 Helm 安装 MySQL 集群
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install mysql-cluster bitnami/mysql \
--set auth.rootPassword=root123 \
--set auth.replicationPassword=repl123 \
--set architecture=replication \
--set primary.persistence.size=10Gi \
--set secondary.replicaCount=2
```
##### 2.2 Redis 集群（哨兵模式）
```bash
helm install redis bitnami/redis \
  --set auth.password=redis123 \
  --set sentinel.enabled=true \
  --set cluster.enabled=false
```
##### 2.3 Nacos 注册中心
```bash
# 使用 Helm 安装 Nacos
helm repo add nacos https://nacos.io/charts
helm install nacos nacos/nacos \
  --set mode=cluster \
  --set service.type=NodePort \
  --set mysql.enabled=true \
  --set mysql.host=mysql-cluster-primary \
  --set mysql.username=root \
  --set mysql.password=root123 \
  --set mysql.database=nacos
```
##### 2.4 RocketMQ
```bash
helm install rocketmq apache/rocketmq \
  --set nameServer.replicaCount=2 \
  --set broker.replicaCount=2
```

#### 3. 微服务模块部署
##### 3.1 代码克隆与编译
```bash
git clone https://github.com/your-repo/hammer-blog.git
cd hammer-blog

# 编译并打包镜像（需提前配置 Docker Hub 或私有仓库）
mvn clean package -DskipTests
docker build -t your-docker-repo/user-service:1.0.0 ./user-service
docker push your-docker-repo/user-service:1.0.0
# 重复上述步骤编译其他服务（article-service、comment-service等）
```
##### 3.2 Kubernetes 资源配置
创建 deployment.yaml 示例（以用户服务为例）：
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: your-docker-repo/user-service:1.0.0
        env:
        - name: NACOS_SERVER_ADDR
          value: "nacos:8848"
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```
##### 3.3 部署服务
```bash
kubectl apply -f deployment.yaml  # 依次部署所有服务
```
#### 4. 配置中心与限流规则
##### 4.1 Nacos 配置管理
通过 Nacos 控制台（访问 http://<nacos-node-ip>:8848/nacos）添加配置：
- Data ID: user-service-dev.yaml
- 配置内容:
```bash
- spring:
  datasource:
    url: jdbc:mysql://mysql-cluster-primary:3306/blog_db?useSSL=false
    username: root
    password: root123
```
##### 4.2 Sentinel 规则持久化
在 Sentinel Dashboard 中配置规则，并同步到 Nacos（需集成 Sentinel-Nacos-Datasource）。

#### 5. 前端部署
# 构建 Vue 项目
```bash
cd hammer-blog-frontend
npm install
npm run build

# 将 dist 目录内容部署到 Nginx
docker build -t your-docker-repo/blog-frontend:1.0.0 .
kubectl apply -f frontend-deployment.yaml
```

#### 6. CI/CD 自动化（可选）
   使用 Jenkins 创建 Pipeline：
```bash
pipeline {
  agent any
  stages {
    stage('Build') {
      steps {
        sh 'mvn clean package -DskipTests'
      }
    }
    stage('Docker Build') {
      steps {
        sh 'docker build -t your-repo/${JOB_NAME}:${BUILD_NUMBER} .'
      }
    }
    stage('Deploy to k8s') {
      steps {
        sh 'kubectl set image deployment/user-service user-service=your-repo/${JOB_NAME}:${BUILD_NUMBER}'
      }
    }
  }
}
```
#### 7. 系统验证
#####    7.1 访问前端
```bash
kubectl get svc blog-frontend  # 获取 NodePort 或 LoadBalancer IP
```
#### 7.2 检查服务状态
```bash
kubectl get pods -n default  # 查看所有 Pod 状态
kubectl logs -f <pod-name>    # 查看日志
```
   
#### 7.3 压力测试
```bash
# 使用 JMeter 对 API 进行压测
jmeter -n -t hammer-blog.jmx -l result.jtl
```
**注意事项**
- 镜像仓库权限：确保 Docker 镜像仓库已授权。
- 持久化存储：MySQL、Redis 等需配置 PVC（PersistentVolumeClaim）。
- 网络策略：按需配置 k8s NetworkPolicy 限制服务间通信。
- 备份与恢复：定期备份数据库（如使用 Velero）。
- 完成以上步骤后，系统即可在 k8s 集群中运行，具备高可用、弹性伸缩和自动化运维能力。

#### 参与贡献
1.  Fork 本仓库
2.  新建 Feat_xxx 分支
3.  提交代码
4.  新建 Pull Request

