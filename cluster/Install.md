# 注意
````
  之前的版本写command和args的时候，格式为： sh,,,-c,,,sleep 36000
  新版写法为：
  sh,,,
  -c,,,
  sleep 36000
  也就是把换行从,,,改成了,,, + 回车，为了兼容deployment的|+ 和 |-
  对应的deployment Command为：
  command:
  - sh
  - -c
  - sleep 36000
````

## 1. 安装Ratel

### 1.1 安装说明

````
    集群安装配置需要两类文件: servers.yaml和集群管理的kubeconfig文件
    
    servers.yaml是ratel的配置文件, 格式如下:
        - serverName: 'xiqu'
          serverAddress: 'https://1.1.1.1:8443'
          #serverAdminUser: 'xxx'
          #serverAdminPassword: 'xxx#'
          serverAdminToken: 'null'
          serverDashboardUrl: "https://k8s.xxx.com.cn/#"
          production: 'false'
          kubeConfigPath: "/mnt/xxx.config"
          harborConfig: "HarborUrl, HarborUsername, HarborPassword, HarborEmail"
        其中管理的方式有两种(Token暂不支持): 
            账号密码和kubeconfig形式, 只需配置一种即可, kubeconfig优先级高

    参数解析:
        serverName: 集群别名
        serverAddress: Kubernetes APIServer地址
        serverAdminUser: Kubernetes管理员账号(需要配置basic auth)
        serverAdminPassword: Kubernetes管理员密码
        serverAdminToken: Kubernetes管理员Token // 暂不支持
        serverDashboardUrl: Kubernetes官方dashboard地址，1.x版本需要添加/#!，2.x需要添加/#
        kubeConfigPath: Kubernetes kube.config路径(绝对路径)
        harborConfig: 对于多集群管理的情况下，可能会存在不同的harbor仓库，配置此参数可以在拷贝资源的时候自动替换harbor配置
    kubeConfigPath 通过secret挂载到容器的/mnt目录或者其他目录

    本文档是将Ratel安装在Kubernetes集群，如果没有Kubernetes集群，可以参考本人写的另一篇文章，CentOS 8二进制高可用安装Kubernetes集群: https://www.cnblogs.com/dukuan/p/11780729.html
````

### 1.2 创建Secret

````
    假设配置两个集群，对应的kubeconfig是test1.config和test2.config
    ratel配置文件servers.yaml内容如下:
        - serverName: 'test1'
          serverAddress: 'https://1.1.1.1:8443'
          #serverAdminUser: 'xxx'
          #serverAdminPassword: 'xxx#'
          serverAdminToken: 'null'
          serverDashboardUrl: "https://k8s.test1.com.cn/#"
          production: 'false'
          kubeConfigPath: "/mnt/test1.config"
          harborConfig: "HarborUrl, HarborUsername, HarborPassword, HarborEmail"
        - serverName: 'test2'
          serverAddress: 'https://1.1.1.2:8443'
          #serverAdminUser: 'xxx'
          #serverAdminPassword: 'xxx#'
          serverAdminToken: 'null'
          serverDashboardUrl: "https://k8s.test2.com.cn/#!"
          production: 'false'
          kubeConfigPath: "/mnt/test2.config"
          harborConfig: "HarborUrl, HarborUsername, HarborPassword, HarborEmail"
    创建Secret: 
        kubectl create secret generic ratel-config  --from-file=test1.config --from-file=test2.config --from-file=servers.yaml -n kube-system
````

### 1.3 创建RBAC
````
创建权限管理namespace
kubectl create ns kube-users

然后添加如下的ClusterroleBinding
vim ratel-rbac.yaml

apiVersion: v1
items:
- apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    annotations:
      rbac.authorization.kubernetes.io/autoupdate: "true"
    labels:
      kubernetes.io/bootstrapping: rbac-defaults
      rbac.authorization.k8s.io/aggregate-to-edit: "true"
    name: ratel-namespace-readonly
  rules:
  - apiGroups:
    - ""
    resources:
    - namespaces
    verbs:
    - get
    - list
    - watch
  - apiGroups:
    - metrics.k8s.io
    resources:
    - pods
    verbs:
    - get
    - list
    - watch
- apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: ratel-pod-delete
  rules:
  - apiGroups:
    - ""
    resources:
    - pods
    verbs:
    - get
    - list
    - delete
- apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: ratel-pod-exec
  rules:
  - apiGroups:
    - ""
    resources:
    - pods
    - pods/log
    verbs:
    - get
    - list
  - apiGroups:
    - ""
    resources:
    - pods/exec
    verbs:
    - create
- apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    annotations:
      rbac.authorization.kubernetes.io/autoupdate: "true"
    name: ratel-resource-edit
  rules:
  - apiGroups:
    - ""
    resources:
    - configmaps
    - persistentvolumeclaims
    - services
    - services/proxy
    verbs:
    - patch
    - update
  - apiGroups:
    - apps
    resources:
    - daemonsets
    - deployments
    - deployments/rollback
    - deployments/scale
    - statefulsets
    - statefulsets/scale
    verbs:
    - patch
    - update
  - apiGroups:
    - autoscaling
    resources:
    - horizontalpodautoscalers
    verbs:
    - patch
    - update
  - apiGroups:
    - batch
    resources:
    - cronjobs
    - jobs
    verbs:
    - patch
    - update
  - apiGroups:
    - extensions
    resources:
    - daemonsets
    - deployments
    - deployments/rollback
    - deployments/scale
    - ingresses
    verbs:
    - patch
    - update
- apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: ratel-resource-readonly
  rules:
  - apiGroups:
    - ""
    resources:
    - configmaps
    - endpoints
    - persistentvolumeclaims
    - pods
    - replicationcontrollers
    - replicationcontrollers/scale
    - serviceaccounts
    - services
    verbs:
    - get
    - list
    - watch
  - apiGroups:
    - ""
    resources:
    - bindings
    - events
    - limitranges
    - namespaces/status
    - pods/log
    - pods/status
    - replicationcontrollers/status
    - resourcequotas
    - resourcequotas/status
    verbs:
    - get
    - list
    - watch
  - apiGroups:
    - ""
    resources:
    - namespaces
    verbs:
    - get
    - list
    - watch
  - apiGroups:
    - apps
    resources:
    - controllerrevisions
    - daemonsets
    - deployments
    - deployments/scale
    - replicasets
    - replicasets/scale
    - statefulsets
    - statefulsets/scale
    verbs:
    - get
    - list
    - watch
  - apiGroups:
    - autoscaling
    resources:
    - horizontalpodautoscalers
    verbs:
    - get
    - list
    - watch
  - apiGroups:
    - batch
    resources:
    - cronjobs
    - jobs
    verbs:
    - get
    - list
    - watch
  - apiGroups:
    - extensions
    resources:
    - daemonsets
    - deployments
    - deployments/scale
    - ingresses
    - networkpolicies
    - replicasets
    - replicasets/scale
    - replicationcontrollers/scale
    verbs:
    - get
    - list
    - watch
  - apiGroups:
    - policy
    resources:
    - poddisruptionbudgets
    verbs:
    - get
    - list
    - watch
  - apiGroups:
    - networking.k8s.io
    resources:
    - networkpolicies
    verbs:
    - get
    - list
    - watch
  - apiGroups:
    - metrics.k8s.io
    resources:
    - pods
    verbs:
    - get
    - list
    - watch
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
  
kubectl create -f ratel-rbac.yaml

vim ratel-rbac-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ratel-namespace-readonly-sa
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ratel-namespace-readonly
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:serviceaccounts:kube-users
  
  kubectl create -f ratel-rbac-binding.yaml


````

### 1.4 部署ratel

````
    ratel的部署文件内容如下:
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          labels:
            app: ratel
          name: ratel
          namespace: kube-system
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: ratel
          strategy:
            rollingUpdate:
              maxSurge: 1
              maxUnavailable: 0
            type: RollingUpdate
          template:
            metadata:
              creationTimestamp: null
              labels:
                app: ratel
            spec:
              containers:
                - command:
                    - sh
                    - -c
                    - ./ratel -c /mnt/servers.yaml
                  env:
                    - name: TZ
                      value: Asia/Shanghai
                    - name: LANG
                      value: C.UTF-8
                    - name: ProRunMode
                      value: prod
                    - name: ADMIN_USERNAME
                      value: admin
                    - name: ADMIN_PASSWORD
                      value: ratel_password
                  image: registry.cn-beijing.aliyuncs.com/dotbalo/ratel:latest
                  imagePullPolicy: Always
                  livenessProbe:
                    failureThreshold: 2
                    initialDelaySeconds: 10
                    periodSeconds: 60
                    successThreshold: 1
                    tcpSocket:
                      port: 8888
                    timeoutSeconds: 2
                  name: ratel
                  ports:
                    - containerPort: 8888
                      name: web
                      protocol: TCP
                  readinessProbe:
                    failureThreshold: 2
                    initialDelaySeconds: 10
                    periodSeconds: 60
                    successThreshold: 1
                    tcpSocket:
                      port: 8888
                    timeoutSeconds: 2
                  resources:
                    limits:
                      cpu: 500m
                      memory: 512Mi
                    requests:
                      cpu: 500m
                      memory: 512Mi
                  volumeMounts:
                    - mountPath: /mnt
                      name: ratel-config
              dnsPolicy: ClusterFirst
              imagePullSecrets:
                - name: myregistrykey
              restartPolicy: Always
              schedulerName: default-scheduler
              securityContext: {}
              terminationGracePeriodSeconds: 30
              volumes:
                - name: ratel-config
                  secret:
                    defaultMode: 420
                    secretName: ratel-config

    需要更改的内容如下:
        ProRunMode: 区别在于dev模式打印的是debug日志, 其他模式是info级别的日志, 实际使用时应该配置为非dev
        ADMIN_USERNAME: ratel自己的管理员账号
        ADMIN_PASSWORD: ratel自己的管理员密码
        实际使用时账号密码应满足复杂性要求,因为ratel可以直接操作所有配置的资源。
        其他无需配置, 端口配置暂不支持。
````

### 1.5 Service和Ingress配置

#### 注意：如果没有安装ingress controller，需要把type: ClusterIP改成type: NodePort，然后通过主机IP+Port进行访问
````
    创建ratel Service的文件如下:
        apiVersion: v1
        kind: Service
        metadata:
          labels:
            app: ratel
          name: ratel
          namespace: kube-system
        spec:
          ports:
            - name: container-1-web-1
              port: 8888
              protocol: TCP
              targetPort: 8888
          selector:
            app: ratel
          type: ClusterIP

    创建ratel Ingress:
        apiVersion: extensions/v1beta1
        kind: Ingress
        metadata:
          name: ratel
          namespace: kube-system
        spec:
          rules:
          - host: krm.test.com
            http:
              paths:
              - backend:
                  serviceName: ratel
                  servicePort: 8888
                path: /
````

### 1.6 访问ratel

#### 注意：如果没有安装ingress controller，需要把type: ClusterIP改成type: NodePort，然后通过主机IP+Port进行访问
````
    通过Ingress配置的krm.test.com/ratel访问，ratel登录页如下:
````
![login](https://github.com/dotbalo/ratel-doc/blob/master/images/login.png)

````
    ratel首页如下:
````

![index](https://github.com/dotbalo/ratel-doc/blob/master/images/index.png)
![index2](https://github.com/dotbalo/ratel-doc/blob/master/images/index2.png)

### 1.7 资源列表

````
    Node列表及配置
    开启驱逐是配置了一个NoExecute的Taint，禁止调度室配置了一个NoSchedule的Taint，并非强制性的。
````

![nodeList](https://github.com/dotbalo/ratel-doc/blob/master/images/nodelist.png)

````
    Deployment StatefulSet DaemonSet列表
    可以直接进行删除、更新(不更新任何内容触发更新)、编辑、添加Service，自动配置Service端口
    点击副本数可以直接进行扩容
````

![stslist](https://github.com/dotbalo/ratel-doc/blob/master/images/stslist.png)

![dslist](https://github.com/dotbalo/ratel-doc/blob/master/images/dslist.png)

![dplist](https://github.com/dotbalo/ratel-doc/blob/master/images/dplist.png)

````
    Service列表，可以针对指定Service添加域名，自动配置端口
````

![svclist](https://github.com/dotbalo/ratel-doc/blob/master/images/svclist.png)

````
    Pod列表，如果状态不是Running，可以将鼠标放在状态上即可查看出错原因
````

![podslist](https://github.com/dotbalo/ratel-doc/blob/master/images/podslist.png)


