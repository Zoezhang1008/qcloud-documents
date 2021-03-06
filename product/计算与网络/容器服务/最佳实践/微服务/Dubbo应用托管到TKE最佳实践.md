## 1. 最佳实践概述

### 1.1 场景描述

本文章介绍了 Dubbo 应用托管到腾讯云容器服务 TKE 的最佳实践。

### 1.2 Dubbo 应用托管到 TKE 的优势

- 提升资源利用率
- Kubernetes 天然适合微服务架构
- 提升运维效率，便于 Devops 落地实施
- Kubernetes 的高弹性，可轻松实现应用的动态扩缩容
- TKE 提供了 Kubernetes Master 托管功能，减少了 Kubernetes 集群运维和管理的负担
- TKE 和腾讯云的其它云原生产品做了整合和优化，帮助用户更好的使用腾讯云上产品

## 2 最佳实践实例 QCBM 介绍

### 2.1 QCBM 概述

本最佳实践以Q云书城（QCBM：Q Cloud Book Mall）项目为例，详细介绍 dubbo 应用托管到 TKE 的整个过程。QCBM 首页如下图展所示：

![](https://main.qcloudimg.com/raw/a6350375839c1eac09b57e2ea4734b6c.png)

QCBM 是一个采用微服务架构，并使用 dubbo-2.7.8 框架开发的一个网上书城 Demo 项目。QCBM 包括如下微服务：

- **QCBM-Front** ：使用 react 开发的前端项目，基于 nginx 官方提供的 [1.19.8 docker 镜像](https://hub.docker.com/_/nginx) 构建和部署。
- **QCBM-Gateway** ：API 网关，接受前端的 http 请求，并转化为后台的 dubbo 请求。
-  **User-Service** ：基于 dubbo 的微服务，提供用户注册、登录、鉴权等功能。
-  **Favorites-Service** ：基于 dubbo 的微服务，提供用户图书收藏功能。
-  **Order-Service** ：基于 dubbo 的微服务，提供用户订单生成和查询等功能。
-  **Store-Service** ：基于 dubbo 的微服务，提供图书信息的存储等功能。

Q云书城的部署和代码托管在 coding 上，对应的地址为 <https://github.com/TencentCloud/container-demo/tree/main/dubbo-on-tke>

![](https://main.qcloudimg.com/raw/8ace423968a9e0b8281245596aed8301.png)

### 2.2 QCBM 架构和组件

最佳实践实例模拟将原先部署在 CVM 上的应用容器化并托管到 TKE 的场景。采用一个 VPC，并划分为两个子网：

- Subnet-Basic 中部署了有状态的基础服务，包括 Dubbo 的服务注册中心 Nacos，MySQL 和 Redis 等。
- Subnet-K8S 中部署 QCBM 的应用服务，所有服务都进行了容器化，运行在 TKE 上；

![](https://main.qcloudimg.com/raw/2868c02edc3ba2df75295a7e15dfd8e4.png)

QCBM 实例的网络规划如下表所示：

| 网络规划              | 说明    |
| :--------------| :---- | 
| Region / AZ          | 南京 / 南京一区    |
| VPC                     | CIDR：10.0.0.0/16 |
| 子网 Subnet-Basic | 南京一区，CIDR：10.0.1.0/24 |
| 子网 Subnet-K8S   | 南京一区，CIDR：10.0.2.0/24 |
| Nacos 集群           | 采用 3 台 “标准型SA2“ 1C2G 机型的 CVM 构建 Nacos 集群，对应的 IP 为：10.0.1.9，10.0.1.14，10.0.1.15 |

QCBM 实例中用到的组件如下表所示：

| 组件  | 版本   | 来源       | 备注       |
|:---- |:------:|:------:|:------|
| K8S  | 1.8.4  | 腾讯云    | TKE 托管模式 |
| MySQL  |  5.7         | 腾讯云    | TencentDB for MySQL 双节点 |
| Redis    |  5.0         | 腾讯云    | TencentDB for Redis 标准型 |
| CLS      |  N/A        | 腾讯云    | 日志服务 |
| TSW      |  N/A       | 腾讯云    | 采用 Skywalking 8.4.0 版的 agent 接入，下载地址: <https://apache.website-solution.net/skywalking/8.4.0/apache-skywalking-apm-es7-8.4.0.tar.gz> |
| JAVA    | 1.8           | 开源社区  | Docker 镜像：java:8-jre |
| Nacos   | 2.0.0       | 开源社区  | 下载链接：<https://github.com/alibaba/nacos/releases/download/2.0.0/nacos-server-2.0.0.tar.gz> |
| Dubbo   | 2.7.8       | 开源社区  | Github地址：Github地址：<https://github.com/apache/dubbo> |


## 3 基础服务集群搭建

- 在 [Mysql 控制台](https://console.cloud.tencent.com/cdb) 创建好实例, 并使用 [qcbm-ddl.sql](https://tencent-cloud-native.coding.net/public/qcbm-k8s/qcbm-k8s/git/files/master/qcbm-ddl.sql) 初始化；
- 在 [Redis 控制台](https://console.cloud.tencent.com/redis) 创建好实例并初始化；
- 在 [CLB 控制台](https://console.cloud.tencent.com/clb) 为子网 Subnet-K8S 新建个内网型的 CLB（后面会用到该 CLB 实例 ID）。
- 部署 Nacos 集群，需要现在 [CVM 控制台](https://console.cloud.tencent.com/cvm) 购买3台 “标准型SA2“ 1C2G 的 CVM，接着安装 java。

   ~~~sh
   # 安装 java
   yum install java-1.8.0-openjdk.x86_64
   # 执行下面命令，如有输出 java 版本信息，说明 java 安装成功
   java -version
   ~~~
   
  最后，部署 Nacos 集群，可参考 Nacos 官方文档  [集群部署说明](https://nacos.io/zh-cn/docs/cluster-mode-quick-start.html) 。


## 4. 构建 docker 镜像

### 4.1 编写 Dockerfile

下面以 user-service 为例简单介绍下 Dockerfile 的编写。下面展示的是 user-service 的工程目录结构，**Dockerfile** 位于工程的根目录下，**user-service-1.0.0.zip** 是打包后的文件，需要添加到镜像中的。

~~~sh
➜  user-service tree
├── Dockerfile
├── assembly
│    ....
├── bin
│    ....
├── pom.xml
├── src
│    ....
├── target
│    .....
│   └── user-service-1.0.0.zip
└── user-service.iml
~~~

user-service 的 Dockerfile 如下：
 
~~~docker
FROM java:8-jre

ARG APP_NAME=user-service
ARG APP_VERSION=1.0.0
ARG FULL_APP_NAME=${APP_NAME}-${APP_VERSION}

# 容器中的工作目录为 /app
WORKDIR /app

# 将本地打包出来的应用添加到镜像中
COPY ./target/${FULL_APP_NAME}.zip .

# 创建日志目录 logs，解压并删除原始文件和解压后的目录
RUN mkdir logs \
    && unzip ${FULL_APP_NAME}.zip \
    && mv ${FULL_APP_NAME}/** . \
    && rm -rf ${FULL_APP_NAME}*

# user-service 的启动脚本和参数
ENTRYPOINT ["/app/bin/user-service.sh"]CMD ["start", "-t"]

# dubbo 端口号
EXPOSE 20880
~~~

> **【Tips】:**
> 
> - 生产中的 java 应用有很多配置参数，导致启动脚本很复杂。因而，将启动脚本里的内容全部写到 dockerfile 中工作量很大，其外 dockerfile 远没有 shell 脚本灵活，出了问题也不好定位，所以不建议弃用启动脚本。 
> 
> - 需要注意一点：通常在启动脚本最后使用 **nohup** 启动 java 应用，但这样启动的 deamon 进程会导致容器运行后直接退出。因而 `nohup java ${OPTIONS} -jar user-service.jar > ${LOG_PATH} 2>&1 &`  这种写法有问题，需要改成 `java ${OPTIONS} -jar user-service.jar > ${LOG_PATH} 2>&1`
> 
> - Dockerfile 中每多一个 RUN 命令，生成的镜像就多一层，最佳做法是将这些 RUN 命令合成一条。


### 4.2 镜像构建

TKE 容器服务提供了自动和手工构建方式，文档 [镜像构建](https://cloud.tencent.com/document/product/1141/50337) 中有详细描述。为了展示具体的构建过程，本文采用手工构建方式。

镜像名称需要符合规范 ***ccr.ccs.tencentyun.com/[namespace]/[ImageName]:[镜像版本号]***，其中命名 namespace 是为了方便镜像管理使用，可以按项目取名，本人采用 qcbm 表示 Q云书城 项目下的所有镜像；ImageName 可以包含 subpath 一般用于企业用户多项目情形下。此外，如果本地已构建好了镜像，可以使用 `docker tag` 命令，按命名规范对镜像重命名。

~~~sh
# 推荐的构建方式，可省去二次打 tag 操作
sudo docker build -t ccr.ccs.tencentyun.com/[namespace]/[ImageName]:[镜像版本号]

# 本地构建 user-service 镜像，最后一个 . 表示 Dockerfile 存放在当前目录（user-service）下
➜  user-service docker build -t ccr.ccs.tencentyun.com/qcbm/user-service:1.0.0 .

# 将已存在镜像按命名规范对镜像重命名
sudo docker tag [ImageId] ccr.ccs.tencentyun.com/[namespace]/[ImageName]:[镜像版本号]
~~~

构建完成后，可使用 `docker images` 命令查看本地仓库中的所有镜像。

![](https://main.qcloudimg.com/raw/3ef0c5d4ff3b33f8ddf8cabfda518665.png)


## 5 上传镜像到 TCR

### 5.1 TCR

腾讯云[容器镜像服务 TCR](https://cloud.tencent.com/product/tcr)，提供了个人版和企业版两种镜像仓库。两者区别如下：

- 个人版镜像仓库仅部署在腾讯云广州，企业版在每个地域都有部署；
- 个人版没有服务 SLA 保证。

![](https://main.qcloudimg.com/raw/bbc0eb5bacab4a309a01600e718ac97d.png)


由于 QCBM 只是个 Dubbo 容器化的 demo 项目，因而个人版完全满足需求。但对于企业用户，推荐使用[企业版 TCR](https://console.cloud.tencent.com/tcr)。有关镜像仓库的具体操作，请见文档 [“镜像仓库基本操作”](https://cloud.tencent.com/document/product/1141/41811)。


### 5.2 创建命名空间

QCBM 项目采用个人版镜像仓库（对于企业客户建议使用企业版镜像仓库）。在 [容器服务控制台](https://console.cloud.tencent.com/tke2) 下的 【镜像仓库】->【个人版】-> 【命名空间】新建一个命名空间 qcbm。QCBM 项目所有的镜像都存放于该命名空间下。

![](https://main.qcloudimg.com/raw/faf9ffeb0c21b3a9f407de6693452768.png)


### 5.3 上传镜像


镜像上传，需要两个步骤：登录腾讯云 registry 和 上传镜像，下面详细介绍 qcbm 镜像的上传过程。


- **Step 1.  登录腾讯云registry**

  ~~~sh
  docker login --username=[腾讯云账号 ID] ccr.ccs.tencentyun.com
  ~~~

  腾讯云账号 ID 可在 [账号信息](https://console.cloud.tencent.com/developer) 页面获取。若忘记 **镜像仓库登录密码**，可在下图所示的容器服务控制台中重置。

	![](https://main.qcloudimg.com/raw/bbb8c32856db3fdc406b70b46a5b27cf.png)

  若提示无权限，请加上 sudo 执行上述命令时，需要输入两个密码：第一个为 sudo 所需的主机管理员密码；第二个为 **镜像仓库登录密码**。

  ~~~sh
  sudo docker login --username=[腾讯云账号 ID] ccr.ccs.tencentyun.com
  ~~~

	![](https://main.qcloudimg.com/raw/d34997020efabeb1f52f3eb9327f20cb.png)


- **Step 2：上传镜像**

  本地生成好的镜像，使用下面命令即可上传到 TKE 的镜像仓库中。

  ~~~sh
  docker push ccr.ccs.tencentyun.com/[namespace]/[ImageName]:[镜像版本号]
  ~~~

	![](https://main.qcloudimg.com/raw/466adcd0ebf9adf2c16421885a0c6567.png)

在 [我的镜像](https://console.cloud.tencent.com/tke2/registry/user/self?rid=1) 中可以查看上传的所有镜像，下图展示的是上传到腾讯云镜像仓库中 QCBM 的 5 个镜像。

![](https://main.qcloudimg.com/raw/e0594eefc368e2cc40282dfaf44c6cee.png)

TCR 镜像默认镜像为“私有”，若想供任何人使用，可在【镜像信息】中设置为公有。

![](https://main.qcloudimg.com/raw/e8cd48462cd76cdc819dfda263ca3b8c.png)


## 6. 在TKE 上部署服务

### 6.1 创建 k8s 集群 qcbm

实际部署前，需要新建个 K8S 集群。有关集群的创建，官方文档 [购买容器集群](https://cloud.tencent.com/document/product/457/9082) 中有详细说明。但有一点需要注意：在创建集群第二步 “选择机型” 时，建议开启 “置放群组功能” 可将 CVM 打散到到不同母机上，增加系统可靠性。

![](https://main.qcloudimg.com/raw/e02eb656cd91db18eb58eabf34b0da69.png)

创建完成后，在容器服务控制台的 [集群管理](https://console.cloud.tencent.com/tke2/cluster) 页面可以看到新建的集群信息。这里我们新建的集群名称为 qcbm-k8s-demo。

![](https://main.qcloudimg.com/raw/bd56002ab91548603dcb3e3e107511a2.png)

选择集群 qcbm-k8s-demo 点击进入 **基本信息** 页面，可以看到整个集群的配置信息。

![](https://main.qcloudimg.com/raw/8b9eba2fa4bf59664cca8a2bbba6acbf.png)

如果需要使用 kubectl 和 lens 等 K8S 管理工具，还需要以下两步操作：

- 开启外网访问
- 将 api 认证 Token 保存为本地 `用户 home/.kube` 下的 config 文件中（若 config 文件已有内容，需要替换），这样每次访问都能进入默认集群中。当然也可以不保存为 `.kube` 下的 config 文件中，相关操作指引见  **集群APIServer信息** 下的 **通过Kubectl连接Kubernetes集群操作说明**。

![](https://main.qcloudimg.com/raw/ab6f61392176377f03d8c2124e2fcd91.png)


### 6.2 创建 Namespace

Namespaces 是 Kubernetes 在同一个集群中进行逻辑环境划分的对象， 可以通过 Namespaces 进行管理多个团队多个项目的划分。在[容器服务控制台](https://console.cloud.tencent.com/tke2) ->【集群】->【命名空间】下创建名称为 qcbm 的Namespace；

更快捷也是推荐的方法是在本地使用命令 kubectl 命令创建：

- 方式 1：直接使用命令行

  ~~~sh
  kubectl create namespace qcbm
  ~~~

- 方式 2：使用 yaml 部署

   ~~~sh
   kubctl create –f namespace.yaml
  ~~~

  其中 namespace.yaml 如下： 

  ~~~yaml
  # 创建命名空间 qcbm
  apiVersion: v1
  kind: Namespace
  metadata:
    name: qcbm
  spec:
    finalizers:
    - kubernetes
  ~~~

### 6.3 使用 ConfigMap 存放配置信息

通过 ConfigMap 可以将配置和运行的镜像进行解耦，使得应用程序有更强的移植性。QCBM 后端服务需要从环境变量中获取 Nacos、MySQL、Redis 的主机和端口信息，这里使用 ConfigMap 来保存。
下面是 QCBM 的 ConfigMap yaml，需要注意一点：纯数字类型的 value 需要用引号括起来，比如下面 yaml 中的 MYSQL_PORT。

~~~yaml
# 创建 ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: qcbm-env
  namespace: qcbm
data:
  NACOS_HOST: 10.0.1.9
  MYSQL_HOST: 10.0.1.13
  REDIS_HOST: 10.0.1.16
  NACOS_PORT: "8848"
  MYSQL_PORT: "3306"
  REDIS_PORT: "6379"
  SW_AGENT_COLLECTOR_BACKEND_SERVICES: xxx   # TSW 接入地址，后文介绍
~~~

此外，也可以手工在[容器服务控制台](https://console.cloud.tencent.com/tke2) ->【集群】 ->【配置管理】-> ConfigMap 下创建名称为 qcbm-env 的 ConfigMap 用于存放相关配置。创建是选择命名空间 qcbm，如下图:

![](https://main.qcloudimg.com/raw/06d5f4de2f5e745ff4750ee1c3826186.png)


### 6.4 使用 Secret 存放敏感信息

Secret 可用于存储密码、令牌、密钥等敏感信息，降低直接对外暴露的风险。QCBM 使用 Secret 来保存相关的账号和密码信息。下面是为 QCBM 创建 Secret 的 yaml。使用 yaml 时需要注意一点：Secret 的 value 需要是 base64 编码后的字符串。

~~~yaml
# 创建 Secret
apiVersion: v1
kind: Secret
metadata:
  name: qcbm-keys
  namespace: qcbm
  labels:
    qcloud-app: qcbm-keys
data:
  # xxx 为base64 编码后的字符串，可使用 shell 命令 “echo -n 原始字符串 | base64” 生成
  MYSQL_ACCOUNT:  xxx
  MYSQL_PASSWORD: xxx
  REDIS_PASSWORD: xxx
  SW_AGENT_AUTHENTICATION: xxx  # TSW 接入 token，后文介绍
type: Opaque
~~~
  
此外，也可以手工在[容器服务控制台](https://console.cloud.tencent.com/tke2) ->【集群】 ->【配置管理】-> Secret 下创建名称为 **qcbm-keys**  的 secret，如下图所示。

![](https://main.qcloudimg.com/raw/3fb95dae4f4afcb758cafc56a939ef89.png)

### 6.5 部署工作负载 Deployment

Deployment 声明了 Pod 的模板和控制 Pod 的运行策略，适用于部署无状态的应用程序。QCBM 的 front 和 Dubbo 服务都属于无状态应用，适合使用 Deployment。

下面是 user-service Deployment 的 yaml，有几点需要说明：

- replicas：表示需要创建的 pod 数量；
- image：镜像的地址；
- imagePullSecrets：拉取镜像时需要使用的key，在【集群】->【配置管理】->【Secret】里可以找到。使用公共镜像时，可省略。
- env：定义了 pod 的环境变量和取值，ConfigMap 中定义的 key-value 可使用 configMapKeyRef 引用；Secret 中定义的 key-value 可使用 secretKeyRef 引用。
- ports：指定容器的端口号，由于是 dubbo 应用，所以为 20880

~~~yaml
# user-service Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: qcbm
  labels:
    app: user-service
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: user-service
      version: v1
  template:
    metadata:
      labels:
        app: user-service
        version: v1
    spec:
      containers:
        - name: user-service
          image: ccr.ccs.tencentyun.com/qcbm/user-service:1.1.4
          env:
            - name: NACOS_HOST  # dubbo服务注册中心nacos的IP地址
              valueFrom:
                configMapKeyRef:
                  key: NACOS_HOST
                  name: qcbm-env
                  optional: false
            - name: MYSQL_HOST  # Mysql 地址
              valueFrom:
                configMapKeyRef:
                  key: MYSQL_HOST
                  name: qcbm-env
                  optional: false
            - name: REDIS_HOST  # Redis的IP地址
              valueFrom:
                configMapKeyRef:
                  key: REDIS_HOST
                  name: qcbm-env
                  optional: false
            - name: MYSQL_ACCOUNT  # Mysql 账号
              valueFrom:
                secretKeyRef:
                  key: MYSQL_ACCOUNT
                  name: qcbm-keys
                  optional: false
            - name: MYSQL_PASSWORD  # Mysql 密码
              valueFrom:
                secretKeyRef:
                  key: MYSQL_PASSWORD
                  name: qcbm-keys
                  optional: false
            - name: REDIS_PASSWORD  # Redis 密码
              valueFrom:
                secretKeyRef:
                  key: REDIS_PASSWORD
                  name: qcbm-keys
                  optional: false
            - name: SW_AGENT_COLLECTOR_BACKEND_SERVICES  # Skywalking 后端服务地址
              valueFrom:
                configMapKeyRef:
                  key: SW_AGENT_COLLECTOR_BACKEND_SERVICES
                  name: qcbm-env
                  optional: false
            - name: SW_AGENT_AUTHENTICATION    # Skywalking agent 连接后端服务的认证 token
              valueFrom:
                secretKeyRef:
                  key: SW_AGENT_AUTHENTICATION
                  name: qcbm-keys
                  optional: false
          ports:
            - containerPort: 20880 # dubbo 端口号
              protocol: TCP
      imagePullSecrets:   # 拉取镜像时需要使用的 key，QCBM 所有服务的镜像已开放为公共镜像，故此处可省略
        - name: qcloudregistrykey
~~~


### 6.6 部署服务 Service

Kubernetes 的 ServiceTypes 允许指定 Service 类型，默认为 ClusterIP 类型。ServiceTypes 的可取如下值：

- LoadBalancer：提供 公网、VPC、内网 访问；
- NodePort：可以通过 云服务器 IP + 主机端口 访问服务；
- ClusterIP：可以通过 服务名 + 服务端口 访问服务；

对于实际生产系统来说，gateway 需要能在 VPC 或 内网范围内进行访问； front 前端需要能对内/外网提供访问。因而，QCBM 的 gateway 和 front 需要制定 LoadBalancer 类型的 ServiceType。TKE 对 LoadBalancer 模式进行了扩展，通过 Annotation 注解配置 Service，可实现更丰富的负载均衡能力。若使用 **service.kubernetes.io/qcloud-loadbalancer-internal-subnetid** 注解，在 service 部署时，会创建内网类型 CLB。一般都事先创建好 CLB，service 的部署 yaml 中使用注解 **service.kubernetes.io/loadbalance-id** 直接指定，提升部署效率。下面是 qcbm-front service 部署 yaml。

~~~yaml
# 部署 qcbm-front service
apiVersion: v1
kind: Service
metadata:
  name: qcbm-front
  namespace: qcbm
  annotations:
    # Subnet-K8S 子网的 CLB 实例 ID
    service.kubernetes.io/loadbalance-id: lb-66pq34pk
spec:
  externalTrafficPolicy: Cluster
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
  selector: # 将后端服务 qcbm-gateway 和该 Service 进行映射
    app: qcbm-front
    version: v1
  type: LoadBalancer
~~~

### 6.7 部署 Ingress

Ingress 是允许访问到集群内 Service 规则的集合。一般使用 Ingress 提供对外访问，而不直接暴露 Service 。QCBM 项目需要为 qcbm-front 创建 Ingress，对应的 yaml 如下。

~~~yaml
# 部署 qcbm-front ingress
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: front
  namespace: qcbm
  annotations:
    ingress.cloud.tencent.com/direct-access: "false"
    kubernetes.io/ingress.class: qcloud
    kubernetes.io/ingress.extensiveParameters: '{"AddressIPVersion":"IPV4"}'
    kubernetes.io/ingress.http-rules: '[{"host":"qcbm.com","path":"/","backend":{"serviceName":"qcbm-front","servicePort":"80"}}]'
spec:
  rules:
    - host: qcbm.com
      http:
        paths:
          - path: /
            backend:  # 关联到后端服务
              serviceName: qcbm-front
              servicePort: 80
~~~

### 6.8 查看部署结果

至此，已完成 QCBM 在 TKE 上的部署。通过容器服务控制台，在【集群】->【服务与路由】->【Ingress】下可以看到创建的 Ingress，通过 ingress 的 VIP 就可以访问 Q云书城 的页面了。

![](https://main.qcloudimg.com/raw/40688b0da6c948e2f8904b85438143db.png)


## 7 集成 CLS 日志服务

### 7.1 开启容器日志采集功能

容器日志采集功能默认关闭，使用前需要开启，步骤如下：

- 登录 容器服务控制台，选择左侧导航栏中的【集群运维】->【功能管理】。
- 在“功能管理”页面上方选择地域，单击需要开启日志采集的集群右侧的【设置】。

	![](https://main.qcloudimg.com/raw/64bb4cfc746e9eae3863b6b7023d45bf.png)

  在“设置功能”页面，单击日志采集【编辑】，开启日志采集后确认。如下图所示：

	![](https://main.qcloudimg.com/raw/b250e381cc698cf5cc507bc4e5b11b67.png)

### 7.2 创建日志集

日志服务区分地域，为了降低网络延迟，尽可能选择与服务邻近的服务地域创建日志资源。日志资源管理主要分为日志集和日志主题，一个日志集表示一个项目，一个日志主题表示一类服务，单个日志集可以包含多个日志主题。QCBM 部署在南京，故在 [日志服务控制台](https://console.cloud.tencent.com/cls/logset) 选择日志服务区域南京。然后点击“创建日志集” 创建日志集 qcbm-logs。

![](https://main.qcloudimg.com/raw/1aceccac94a3a0a09ee1e4e927701dc0.png)

### 7.3 创建日志主题

在 [日志服务控制台](https://console.cloud.tencent.com/cls/logset) 单击“日志集名称”，进入到日志主题管理页面。单击 “新增日志主题”，开始创建日志主题。QCBM 有多个后端微服务，为每个微服务建个日志主题便于日志归类。

- QCBM 每个服务都建立了一个日志主题。
- 日志主题 ID，为容器创建日志规则时需要用到。

![](https://main.qcloudimg.com/raw/67ec10483e92661504baf04d1b3fcae5.png)

### 7.4 配置日志采集规则

可通过控制台或 CRD 两种方式配置容器日志采集规则。

#### 7.4.1 通过容器服务控制台配置日志采集规则

日志规则指定了日志在容器内的位置，在容器服务控制台左侧导航栏中的【集群运维】->【日志规则】中可新建日志规则。

- 日志源：指定容器日志位置，QCBM 的日志都统一输出到 /app/logs 目录下，因而使用容器文件路径并指定具体的工作负载和日志位置。
- 消费端：选择之前创建的日志集和主题。

![](https://main.qcloudimg.com/raw/5c8882ec907aaaad681cf0f2432e7bba.png)

点击下一步，进入 “日志解析方式”，简单起见 qcbm 使用单行文本方式，这里不再赘述。 CLS 支持的日志格式见文档：<https://cloud.tencent.com/document/product/614/17418> 。

#### 7.4.2 通过 CRD 创建容器日志采集规则

用户还可通过自定义资源定义（CustomResourceDefinitions，CRD）的方式配置日志采集。QCBM 使用了容器文件路径的采集方式，日志格式为单行文本，下面是 user-service 日志采集具体的配置 yaml。更多的 CRD 采集配置参考：<https://cloud.tencent.com/document/product/457/48425>

~~~yaml
apiVersion: cls.cloud.tencent.com/v1
kind: LogConfig
metadata:
  name: user-log-rule
spec:
  clsDetail:
    extractRule: {}
    # 单行文本
    logType: minimalist_log
    # 日志主题 user-log 的 ID
    topicId: 0c544491-03c9-4ed0-90c5-9bedc0973478
  inputDetail:
    # 日志所在的容器、工作负载以及日志输出目录
    containerFile:
      container: user-service
      filePattern: '*.log'
      logPath: /app/logs
      namespace: qcbm
      workload:
        kind: deployment
        name: user-service
    # 日志采集类型为 容器文件路径
    type: container_file
~~~

### 7.5 查看日志

在[日志服务控制台](https://console.cloud.tencent.com/cls/search)的【检索分析】中可先为日志【新建索引】，让后点击 【检索分析】按钮即可查看日志。未建索引，会看不到日志，这点需要注意。

![](https://main.qcloudimg.com/raw/b14fec221bc6e7c7692d8e4d1fe78cfe.png)


## 8 集成 TSW 观测服务

### 8.1 TSW 简介

腾讯微服务观测平台 TSW（Tencent Service Watcher）提供云原生服务可观察性解决方案，能够追踪到分布式架构中的上下游依赖关系，绘制拓扑图，提供服务、接口、实例、中间件等多维度调用观测。有关 TSW 的详细特性和使用方式，参见：<https://cloud.tencent.com/product/tsw>

![](https://main.qcloudimg.com/raw/9bbb63a967a059098270661a6194dbf0.png)

TSW 在架构上分为以下四大模块：

- 数据采集（Client）：

  使用开源探针或 SDK 用于采集数据。对于迁移上云的用户，可保留 Client 端的大部分配置，仅更改上报地址和鉴权信息即可。

- 数据处理（Server）：

  数据经由 Pulsar 消息队列上报到 Server，同时 Adapter 会将数据转换为统一的 Opentracing 兼容格式。根据数据的使用场景，分配给实时计算与离线计算。实时计算提供实时监控、统计数据展示，并对接告警平台快速响应。离线计算处理长时段大量数据的统计汇聚，利用大数据分析能力提供业务价值。
  
- 存储（Storage）：

  存储层可满足不同数据类型的使用场景，适配 Server 层的写入与 Data Usage 层的查询与读取请求。
  
- 数据使用（Data Usage）：

  为控制台操作、数据展示、告警提供底层支持。

![](https://main.qcloudimg.com/raw/8ad0a1b76bdda3c101cc4505c3952242.png)

### 8.2 接入 TSW — 获取接入点信息

TSW 目前处于内测阶段，可在页面 <https://cloud.tencent.com/apply/p/rvo6c9fnug> 申请内测后使用 TSW 控制台 <https://console.cloud.tencent.com/tsw> 接入服务。目前 TSW 支持 JAVA 和 Golang 两种语言接入。

在 TSW 控制台的【服务观测】->【服务列表】页，单击【接入服务】，选择 Java 语言与 SkyWalking 的数据采集方式。接入方式下提供了如下接入信息：**接入点** 和 **Token**。

![](https://main.qcloudimg.com/raw/b56682adba6bdfb330fe98aa7dbf09fa.png)

>TSW 目前处于内测阶段，只在 广州 和 上海 进行了部署，在写本实例时选择了上海接入（QCBM 部署在南京）。

### 8.3 接入 TSW — 应用和容器配置

将上一节中获取的 TSW 的 接入点 和 Token 分别与填写到 skywalking 的 agent.config 里的配置项 collector.backend_service 和 agent.authentication。“agent.service_name” 配置上对应的服务名称，可使用 “agent.namespace” 对同一领域下的微服务归类。下图是 user-service 的配置。

![](https://main.qcloudimg.com/raw/a6a1818458fd003f8fd826d3b05e2087.png)

Skywalking agent 也支持使用环境变量方式进行配置，QCBM 使用 ConfigMap 和 Secret 配置对应的环境变量：

- 使用 ConfigMap 配置 SW_AGENT_COLLECTOR_BACKEND_SERVICES
- 使用 Secret 配置 SW_AGENT_AUTHENTICATION

![](https://main.qcloudimg.com/raw/5f3a034b41fd6e4df08b9fa2c23ce7c0.png)

至此 TSW 接入工作已完成，启动容器服务后，在 TSW 控制台即可查看调用链、服务拓扑、SQL分析等功能。

### 8.4 使用 TSW 观测服务

#### 8.4.1 通过服务接口和调用链查看调用异常

在 [TSW 控制台](https://console.cloud.tencent.com/tsw)【服务观测】-> 【接口观测】页面可查看一个服务下所有接口的调用情况：请求量、成功率、错误率、响应时间等指标。

![](https://main.qcloudimg.com/raw/05b4fd1dfba7f90152d8423a02e711ea.png)

上图展示了 qcbm-gateway 的两个接口：查询用户收藏夹 `/api/favorites/query/{userId}` 和 查询用户订单 `/api/order/{userId}` 出现调用异常。点击查询用户收藏夹接口，可以看到该接口的所有调用记录，找到异常的调用链，点击进入可以查看具体异常原因。

![](https://main.qcloudimg.com/raw/aa2c92ad7553263f8bfd0c825094cf11.png)

favorites-service 因 time-out 导致调用异常。

![](https://main.qcloudimg.com/raw/08ab1a02001e4cce629fd9b49e330d64.png)


#### 8.4.2 使用 TSW 分析 SQL 和 缓存等组件调用情况

在 TSW 控制台【组件调用观测】页面可查看 SQL、NOSQL、MQ 及其它组件的调用情况。比如，通过 SQL 的请求量及耗时，可以快速定位应用中的高频 SQL 和慢查询。

![](https://main.qcloudimg.com/raw/2959f7dfbad27673ca6e900a78195155.png)

### 8.4.2 查看服务拓扑

在 TSW 控制台【链路追踪】-> 【分布式依赖拓扑】页面可查看完成的服务依赖情况，以及调用次数和平均延迟等信息。

![](https://main.qcloudimg.com/raw/b2cbf6887923a67c9dd993bb5953f7a3.png)
