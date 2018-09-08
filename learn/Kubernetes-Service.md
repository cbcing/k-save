Kubernetes Service

************

### 1、Servcie定义

```yaml
apiVersion: v1
kind: Service
# 元数据
metadata:
  # 自定义标签属性列表
  labels:
    app: mzk-cms-1
  # service名称
  name: mzk-cms-1
  namespace: default
# 详细描述
spec:
  # service需要暴露的端口列表
  ports:
  - port: 17074
    protocol: TCP
    # 需要转发到后端Pod的端口号
    targetPort: 17074
  # 选择具有指定Label标签的Pod作为管理范围
  selector:
    app: mzk-cms-1
  # 是否支持session
  sessionAffinity: None
  # service类型
  type: ClusterIP
status:
  loadBalancer: {}
```



### Service基本用法

##### service的创建

* 直接根据rc创建
```SHELL
kubectl expose rc <rc-name>
```
* 根据service.yaml文件创建
```shell
kubectl create -f service-yaml-name.yaml
```

##### service两种自动负载分发策略
* RoundRobin: 轮询（默认）
* SessionAffinity: 基于客户端IP地址进行会话保持模式

##### 应用系统将一个外部数据库（或另外一个集群或Namespce中服务）作为后端服务进行连接

1. 创建一个无Lable Selector的service
2. 手动创建Endpoint，指向实际的后端访问地址



### 集群外部访问Pod或Service

##### 简述

​	Pod和Service时k8s集群范围的虚拟概念，集群外的客户段系统无法通过Pod和IP地址或者Service的虚拟IP地址和虚拟端口号访问到它们。为了让外部客户段可以访问到这些服务，可以将Pod或者Service的端口号映射到宿主机，是的客户端应用能够通过物理机访问容器应用。

##### 容器应用端口好映射到物理主机

* 设置容器级别的hostPort

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: mzk-cms-1
  namespace: default
  labels:
    app: mzk-cms-1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: mzk-cms-1
    spec:
      nodeSelector:
        app: java
      containers:
      - name: mzk-cms-1
        image: registry-1.docker.io/spring-boot/mzk-cms:v1.0
        ports:
        - containerPort: 17074
          # 将容易应用的端口号17074映射到物理主机8081
          hostPost: 8081
        volumeMounts:
          - name: log
            mountPath: /log
      volumes:
        - name: log
          hostPath:
            path: /data/logs/mzk/17074-cms-1
```

* 设置Pod级别的hostNetwork=true

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: mzk-cms-1
  namespace: default
  labels:
    app: mzk-cms-1
spec:
  # 设置hostNetwork=true，默认物理主机端口号和容器端口号一致
  hostNetwork: true
  replicas: 1
  template:
    metadata:
      labels:
        app: mzk-cms-1
    spec:
      nodeSelector:
        app: java
      containers:
      - name: mzk-cms-1
        image: registry-1.docker.io/spring-boot/mzk-cms:v1.0
        ports:
        - containerPort: 17074
        volumeMounts:
          - name: log
            mountPath: /log
      volumes:
        - name: log
          hostPath:
            path: /data/logs/mzk/17074-cms-1
```



##### Service端口号映射到物理主机

* 设置nodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: mzk-cms-1
  name: mzk-cms-1
  namespace: default
spec:
  # 设置nodePort映射到物理主机
  type: NodePort
  ports:
  - port: 17074
    protocol: TCP
    targetPort: 17074
    # 设置nodePort映射到物理主机
    nodePort: 8081
  selector:
    app: mzk-cms-1
  sessionAffinity: None
status:
  loadBalancer: {}
```

* 设置LoadBalancer

  1、通过LoadBalancer映射到云服务商提供的LoadBalancer地址。

  2、这种方式仅用于在公有云服务提供商的云平台上设置Servcie的场景。

  3、下面这个例子，status.loadBalancer.ingress.ip设置为146.148.47.155为云服务商提供的负载均衡器的ip地址，对该Service的访问请求回通过LoadBalancer转发到后端的Pod上，负载分发的实现方式依赖于云服务商提供的LoadBalancer的实现机制。

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: mzk-cms-1
  name: mzk-cms-1
  namespace: default
spec:
  ports:
  - port: 17074
    protocol: TCP
    targetPort: 17074
    #
    nodePort: 30061
  selector:
    app: mzk-cms-1
  sessionAffinity: None
  #
  clusterIP: 10.0.171.239
  type: LoadBalancer
status:
  #
  loadBalancer:
  	ingress:
  	_ ip: 146.148.47.155
```



### 2、DNS服务搭建指南

略...



### 3、Ingress : HTTP 7层路由机制

* 简述

  ​	对于基于HTTP的服务来说，不同的URL地址经常对应到不同的后端服务或者虚拟机服务器，这些应用层的转发机制仅仅通过Kubernetes的Service机制是无法实现的。

  ​	Kubernetes v1.1版本中新增的Ingress将不同的URL的访问请求转发到后端不同的Servcie，实现HTTP层的业务路由机制。

  ​	在Kubernetes集群中，Ingress的实现需要通过Ingress的定义与Ingress Controller的定义结合起来，才能形成完整的HTTP负载分发功能。

##### 实践操作

* 1、创建Ingress Controller

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-ingress
  labels:
    app: nginx-ingress
spec:
  replicas: 1
  selector:
    app: nginx-ingress
  template:
    metadata:
      labels:
        app: niginx-ingress
      spec:
        containers:
        - image: gcr.io/google_containers/nginx-ingress:0.1
          name: nginx
          ports:
          - containerPort: 80
            hostPort: 80
```

```shell
kubectl create -f nginx-ingress-rc.yaml
```

* 2、定义Ingress

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: mywebsite-ingress
spec:
  rules:
  - host: yourwebsite.com
    http:
      path: /web
      backend:
        serviceName: webapp
        servccePort: 80
```

​	这个Ingress的定义说明了URL http://www.yourwebsite.com/web 的访问将被转发到Kubernetes的一个Servcie上：webapp：80

```shell
kubectl create -f ingerss.yaml
```

