Kubernetes Pod

******

### 目录

1. Pod定义
2. Pod基本用法
3. Pod配置管理
4. Pod生命周期与重启策略
5. Pod健康检查
6. Pod调度
7. Pod扩容与缩容
8. Pod滚动升级



### 1、Pod定义

```shell
kubectl get pod mzk-cms-1-980778657-nj8qd -0 yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubernetes.io/created-by: |
      {"kind":"SerializedReference","apiVersion":"v1","reference":{"kind":"ReplicaSet","namespace":"default","name":"mzk-cms-1-980778657","uid":"6fe8b432-9adb-11e8-8a51-005056889047","apiVersion":"extensions","resourceVersion":"115907504"}}
  creationTimestamp: 2018-08-08T08:02:46Z
  generateName: mzk-cms-1-980778657-
  labels:
    app: mzk-cms-1
    pod-template-hash: "980778657"
  name: mzk-cms-1-980778657-nj8qd
  namespace: default
  ownerReferences:
  - apiVersion: extensions/v1beta1
    controller: true
    kind: ReplicaSet
    name: mzk-cms-1-980778657
    uid: 6fe8b432-9adb-11e8-8a51-005056889047
  resourceVersion: "115914695"
  selfLink: /api/v1/namespaces/default/pods/mzk-cms-1-980778657-nj8qd
  uid: 6f5f1691-9ae1-11e8-8a51-005056889047
spec:
  containers:
  - image: registry-1.docker.io/spring-boot/mzk-cms:v1.0
    imagePullPolicy: IfNotPresent
    name: mzk-cms-1
    ports:
    - containerPort: 17074
      protocol: TCP
    resources: {}
    terminationMessagePath: /dev/termination-log
    volumeMounts:
    - mountPath: /log
      name: log
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-22c18
      readOnly: true
  dnsPolicy: ClusterFirst
  nodeName: node-245-102.server.com
  nodeSelector:
    app: java
  restartPolicy: Always
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  volumes:
  - hostPath:
      path: /data/logs/mzk/17074-cms-1
    name: log
  - name: default-token-22c18
    secret:
      defaultMode: 420
      secretName: default-token-22c18
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: 2018-08-08T08:02:46Z
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: 2018-08-08T08:15:03Z
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: 2018-08-08T08:02:46Z
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: docker://af4f2bb382d50d43afd3ab667770fb04719f77fbfc90c57da9011c3dab37696f
    image: registry-1.docker.io/spring-boot/mzk-cms:v1.0
    imageID: docker-pullable://registry-1.docker.io/spring-boot/mzk-cms@sha256:41ed82820e730d9068ec0d41f4945eaf1cc36d2a3378ab99c5929afdc59b0cb1
    lastState: {}
    name: mzk-cms-1
    ready: true
    restartCount: 0
    state:
      running:
        startedAt: 2018-08-08T08:15:02Z
  hostIP: 10.25.245.102
  phase: Running
  podIP: 172.16.65.2
  startTime: 2018-08-08T08:02:46Z
```



### 2、Pod基本用法

##### Pod可以由一个或者多个容器组合而成

eg.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis-php
  labels:
    name: redis-php
  spec:
    # 容器1
    containers:
    - name: frontend
      images: kubeguide/guestbook-php-frontend:localredis
      ports:
      - containerPort: 80
    # 容器2
    - name: redis
      images: kubeguide/redis-master
      ports:
      - containerPort: 6379
```

同一Pod中的多个容器应用之间通过localhost就可以相互通信了。



### 静态Pod

​	静态Pod是由kubelet进行管理的仅存在于特定Node上的Pod。



### Pod容器共享Volume

* 将容器中的/log目录挂在到主机/data/logs/mzk/17074-cms-1下

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
        # 容器目录
        volumeMounts:
          - name: log
            mountPath: /log
      # 主机目录
      volumes:
        - name: log
          hostPath:
            path: /data/logs/mzk/17074-cms-1
```

* Pod容器共享volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
  - name: tomcat
    image: tomcat
    ports:
    - containerPort: 8080
    # 容器1目录
    valumeMounts:
    - name: app-logs
      mountPath: /usr/local/tomcat/logs
  - name: busybox
    image: busybox
    command: ["sh", "-c", "tail -f /logs/catalina*.log"]
    # 容器2目录
    volumeMounts:
    - name: app-logs
      mountPath: /logs
  # 主机目录 
  volumes:
  - name: app-logs
    emptyDir: {}
```



### 3、Pod配置管理

##### 简述

​	应用配置的一个最佳实践是将应用所需的配置信息与程序进行分离，这样可以使得应用程序被更好的复用，通过不同的配置也能实现更加灵活的功能。将应用打包为容器镜像之后，可以通过环境变量或者外挂文件的方式在创建容器的时候进行配置注入 ，但是在大规模的集群环境中，对于多个容器进行不同的配置将变得非常的复杂。Kubernetes v1.2版本提供了一种同一的集群配置管理方案——ConfigMap

##### ConfigMap

* ConfigMap提供的功能：
  1. 生成为容器内部的环境变量
  2. 设置容器启动命令的启动参数（需要设置为环境变量）
  3. 以Volume的形式挂在为容器内部的文件或目录
* 操作

创建cms-appvars.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: your-appvars
data:
  apploglevel: info
  appdatadir: /var/data
```

创建ConfigMap

```shell
kubectl create -f cms-appvars.yaml
```

查看

```
kubectl get configmap
```

详细信息

```
kubectl descirbe config cms-appvars
```

查看yaml

```
kubectl get config cms-appvars -o yaml
```



##### 创建ConfigMap的方式

* 通过yaml文件

```shell
kubectl create configmap <config-name.yaml>
```

* 直接通过命令行的方式1

```shell
kubectl create configmap <configmap-name> --from-file=[key=]souce
```

通过--from-file参数从文件中创建，也可以指定key的名称。

* 直接通过命令行的方式2

```shell
kubectl create configmap <configmap-name> --from-file=config-files-dir
```

通过--from-file参数从目录中进行创建，改目录下每个配置文件的文件名被设置为key，文件的内容被设置为value

* 直接通过命令行的方式3

```shell
kubectl create configmap <configmap-name> --from-literal=key1=value1 --from-literal-key2=value2 --from-literal=key3=value3
```



##### ConfigMap的使用

##### ConfigMap的使用：环境变量的方式

cms-appvars.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  # configmap 的名称
  name: your-appvars
# 环境变量key-value设置
data:
  apploglevel: info
  appdatadir: /var/data
```

mzk-cms-deploy.yaml

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
        # 环境变量设置
        env:
          # 环境变量名称
        - name: APPLOGLEVEL
          # 环境变量的值来自于configmap
          valueFrom:
            configMapKeyRef:
              # configmap名称
              namd: cms-appvars
              # 该configmap下的key值
              key: apploglevel
          # 同上
        - name: APPDATADIR
          valueFrom:
            configMapKeyRef:
              name: cms-appvars
              key: appdatadir
        ports:
        - containerPort: 17074
          hostPost: 8081
        volumeMounts:
          - name: log
            mountPath: /log
      volumes:
        - name: log
          hostPath:
            path: /data/logs/mzk/17074-cms-1
```



##### ConfigMap的使用：volumeMount模式

略



### 4、Pod生命周期和重启策略

##### Pod状态

| 状态值    | 描述                                                         |
| --------- | ------------------------------------------------------------ |
| Pending   | Pod已经创建，Pod内还存在容器的镜像还没有创建，包括正在下载镜像的过程 |
| Running   | Pod的所有容器均已创建，至少有一个容器处于运行或正在启动或正在重启的状态 |
| Succeeded | Pod所有容器均已成功执行并且退出，且不会在重启                |
| Failed    | Pod内所有容器均已退出，但至少存在一个容器退出状态为失败      |
| Unknown   | 不知道Pod状态                                                |

##### Pod重启策略(RestartPolicy)

| 方式      | 描述                                                 |
| --------- | ---------------------------------------------------- |
| Always    | 当容器失败时，kubelet自动重启该容器                  |
| OnFailure | 当容器中止运行且退出码不为0时，kubelet自动重启该容器 |
| Never     | 无论容器运行状态如何，kebelet都不会重启该容器        |

备注：

1. kubelet为Node上的工具
2. Pod的重启策略与控制方式息息相关



### 5、Pod健康检查

##### 两类探针对Pod进行健康检查

| 探针类型       | 描述                                                         |
| -------------- | ------------------------------------------------------------ |
| LivenessProbe  | 用于判断容器是否存活（running状态），如果容器不健康，kubelet杀掉容器，并且根据容器的重启策略做相应的处理。如果一个容器不包含LivenessProbe探针，kubelet则认为容器的LivenessProbe返回的值时“Success” |
| ReadinessProbe | 用于判断容器是否启动完成（ready状态），并且可以接受请求。如果ReadinessProbe谈征检测到失败，则Pod的状态将被修改。Endpoint Controller将从Service的Endpoint中删除包含该容器所在的Pod的Endpoint |

备注：kubelet定期执行LivenessProbe探针来诊断容器的健康状态。

##### LivenessProbe的实现方式

* ExecAction

  描述：在容器内部执行一个命令，如果该命令的返回码为0，则表明容器健康。

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    labels:
      test: liveness
    name: liveness-exec
  spec:
    containers:
    - name: liveness
      images: gcr.io/google_containers/busybox
      # 设置容器启动参数
      args:
      - /bin/sh
      - -c
      - echo ok > /tmp/health; sleep 10; rm -rf /tmp/health; sleep 600
      # livenessProbe探针设置
      livenessProbe:
        # 执行命令：cat /tmp/health
        exec:
          command:
          - cat
          - /tmp/health
        # 初始探测时间为15秒
        initialDelaySeconds: 15
        timeoutSeconds: 1
  ```

  备注：

  1. 上述容器在启动之后，会创建一个文件/tmp/health，然后在10秒后删除；在容器启动15后会开始进行健康检查，执行命令cat /tmp/health，但是由于在10秒时该文件已被删除，所以命令执行失败，会导致kubelet杀掉该容器并重启它。
  2. timeoutSeconds: 表示健康检查发出请求后的超时时间，单位为秒。当超时发生时，kubelet会认为容器已经无法提供发布，会重启该容器。

  

  额外：yaml to json

  ```json
  {
  	"apiVersion": "v1",
  	"kind": "Pod",
  	"metadata": {
  		"labels": {
  			"test": "liveness"
  		},
  		"name": "liveness-exec"
  	},
  	"spec": {
  		"containers": [
  			{
  				"name": "liveness",
  				"images": "gcr.io/google_containers/busybox",
  				"args": [
  					"/bin/sh",
  					"/-c",
  					"echo ok > /tmp/health; sleep 10; rm -rf /tmp/health; sleep 600"
  				],
  				"livenessProbe": {
  					"exec": {
  						"command": [
  							"cat",
  							"/tmp/health"
  						]
  					},
  					"initialDelaySeconds": 15,
  					"timeoutSeconds": 1
  				}
  			}
  		]
  	}
  }
  ```

* TCPSocketAction

  简述：通过容器内的IP地址和端口号执行TCP检查，如果能够建立TCP连接，则表明容器健康。 

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-with-healthcheck
  spec:
    containers:
    - name: nginx
      images: nginx
      ports:
      - containerPort: 80
      # LivnessProbe探针设置
      livenessProbe:
        tcpSocket:
          port: 80
        initialDelaySeconds: 30
        timeoutSeconds: 1
  ```

  备注：上述例子，启动容器30秒后与容器内的localhost:80建立TCP连接进行健康检查。

* HTTPGetAction

  简述：通过容器内的IP地址，端口号以及路径调用HTTP Get方法，如果相应的状态码大于等于200且小于等于400，则认为容器健康。

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-with-healthcheck
  spec:
    containers:
    - name: nginx
      image: nginx
      ports:
      - containerPort: 80
      # LivenessProbe探针设置
      livenessProbe:
        httpGet:
          path: /_status/healthz
          port: 80
        initialDelaySeconds: 30
        timeuotSeconds: 1
  ```

  备注：启动容器30秒后调用localhost:80/_status/healthz接口进行健康检查。



### 6、Pod调度

#### RC、Deployment：全自动调度

##### NodeSelector：定向调度

* 首先给node打上标签

  ```shell
  kubectl label nodes k8s-node-1 zone=cbc
  ```

* rc.yaml

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
            hostPost: 8081
          volumeMounts:
            - name: log
              mountPath: /log
        volumes:
          - name: log
            hostPath:
              path: /data/logs/mzk/17074-cms-1
    # 定向调度到cbc这个node上
    nodeSelector:
      zone: cbc
  ```

* 查看Pod所在的Node

  ```shell
  kubectl get pods -o wide
  ```

  

##### 亲和性调度(NodeAffinity)

* 简述

  ​	由于NodeSelector通过Node的label进行精确匹配，所以NodeAffinity增加了In（或）、NotIn（不属于）、Exists（存在一个条件）、DoesNotExist（不存在）、Gt（大于）、Lt（等于）等操作符来选择Node，能够使调度策略更加灵活。同时，在NodeAffinity中将增加一些信息来设置亲和性调度策略。

  1. RequiredDuringSchedulingRequireDuringExecution：在Node不满足条件的时候，系统将从Node上移除之前调度上的Pod。
  2. RequiredDuringSchedulingIgnoredDuringExecution：在Node不满足条件的时候，系统不一定从Node上移除之前调度上的Pod。
  3. PreferredDuringSchedulingIgnoredDuringExecution：指定在满足调度条件的Node中，哪些Node应该优先的进行调度。同时在Node不满足条件的时候，系统不一定从该Node上移除之前调度的Pod。

* yaml

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: with-labels
    annotations:
      # 设置NodeAffinity内容
      scheduler.alpha.kubernetes.io/affinity: >
        {
            "nodeAffinity": {
                "requiredDuringSchedulingIgnoreDuringExecution": {
                    "nodeSelectorTerms": [
                        {
                            "matchExpressions":[
                                {
                                    "key": "kubernetes.io/e2e-az-name",
                                    "operator": "In",
                                    "values": ["e2e-az1", "e2e-az2"]
                                }
                            ]
                        }
                    ]
                }
            }
        }
      another-annotation-key: another-annotation-values
    spec:
      containers:
      - name: with-labels
        image: gcr.io/google_containers/pause:2.0
  ```

  备注：

  1. **这里的NodeAffinity的设置说明了只有Node的Label中包含key=kubernetes.io/e2e-az-name并且其value值为"e2e-az1"或"e2e-az2"时，才能成为该Pod的调度目标**。
  2. 如果同时设置了NodeSelector和NodeAffinity，则系统需要同时满足两者的设置才能进行调度。

#### DaemonSet: 特定场景调度

略

#### Job批处理调度

略



### 7、Pod的扩容和缩容

##### 通过命令行的方式

* 将pod数修改为指定的数目

```shell
kubectl scale rc redis-slave --replicas=3
```

备注：

1. 将redis-slave这个pod的副本数目设置为3。
2. 将--replicas设置为比当前副本数量更小的数字，系统会“杀掉”一些运行中的Pod。

##### 通过HPA的方式

* 简述

  ​	Horizontal Pod Autoscaler(HPA)控制器，用于实现基于CPU使用率进行自动Pod扩容或者缩容的功能。HPA控制器基于Master的kube-controller-manager服务启动参数 --horizontal-pod-sync-period定义的时长（默认30秒），周期性的监测目标Pod的CPU使用率，在满足条件的情况下对RC或者Reployment中的Pod副本数量进行调整，以符合用户定义的平均Pod CPU使用率。Pod的CPU使用率来源于heapster组建，需事先安装。

* **给RC/Deployment设置HPA**

  1、deployment.yaml

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
        containers:
        - name: mzk-cms-1
          image: registry-1.docker.io/spring-boot/mzk-cms:v1.0
          # 设置Pod的CPU平均使用率
          resources: 
            requests:
              cpu: 200m
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

  ```shell
  kubectl create -f <deployment-name.yaml>
  ```

  

  2、service.yaml

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
    selector:
      app: mzk-cms-1
    sessionAffinity: None
    type: ClusterIP
  status:
    loadBalancer: {}
  ```

  ```shell
  kubectl create -f <service-name.yaml>
  ```

  3、为上面的Deployment创建一个HPA控制器

  ```shell
  kubectl autoscale deployment <deployment-name.yaml> --min=1 --max=10 --cpu-percent=50
  ```

  描述：在1和10之间调整Pod的副本数量，是的平均Pod CPU使用率维持在50%。

* **通过yaml配置文件创建HPA**

  1、hpa.yaml

  ```yaml
  apiVersion: autoscaling/v1
  kind: HorizontalPodAutoscaler
  # 元数据
  metadata:
    name: demo-hpa
  # 详细信息
  spec:
    # 此HPA的使用目标
    scaleTargetRef:
      apiVersion: v1
      kind: Deployment
      # 使用目标名称
      name: mzk-cms-1
    # 设置目标Pod最小副本值为1
    minReplicas: 1
    # 设置目标Pod最大副本值为10
    maxReplicas: 10
    # 设置目标Pod平均CPU使用率为50%
    targetCPUUtilizationPercentage: 50
  ```

  创建HPA资源

  ```shell
  kubectl create -f demp-hpa.yaml
  ```



### 8、Pod滚动升级

* 简述

  ​	Kubectl rolling-update命令用于滚动升级，它会先创建一个新的RC，然后自动控制旧的RC中的Pod副本数量并且使之逐渐的减少至0，同时新建的RC的Pod数量从0逐渐增加点到目标值，最终实现Pod的全部升级。

  备注：新旧RC必须属于同一个命令空间。

* rc.yaml

  ```yaml
  apiVersion: v1
  kind: ReplicationController
  metadata:
    # 此RC为redis-master的v2版本，也就是需要升级的版本
    name: redis-master-v2
    # 新RC标签设置为和老的一样，只是标签版本为V2
    labels:
      name: redis-master
      vresion: v2
  spec:
    replicas: 1
    selector:
      name: redis-master
      version: v2
    template:
      metadata:
        labels:
          name: redis-master
          version: v2
      spec:
        containers:
        - name: master
          image: kubeguide/redis-master:2.0
          port:
          - containerPorts: 6379
  ```

  执行命令进行滚动升级：

  ```shell
  kubectl rolling-update redis-master -f <your-update-rc-name.yaml>
  ```

* 不使用yaml配置文件进行滚动升级

  ```shell
  kubectl rolling-update redis-master --image=redis-master:2.0
  ```

  