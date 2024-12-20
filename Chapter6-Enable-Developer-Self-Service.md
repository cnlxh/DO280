```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 启用开发人员自助服务

# 课程目标

- 在集群范围内配置每个项目的计算资源配额和 Kubernetes 资源计数配额。​

- 配置每个项目 pod 的默认和最大计算资源要求。​

- 为新项目配置默认配额、​限值范围、​角色绑定和其他限制，以及允许用户自行置备新项目。​

# 项目和集群配额

## 限制工作负载的必要性以及方法

允许用户自行创建工作负载可以提高工作效率，集群的资源有限，如 CPU、​RAM 和存储。​如果集群上的工作负载超出了可用资源，则工作负载可能无法正常工作，为帮助解决这个问题，Kubernetes 工作负载可以预留资源并声明资源限值。​工作负载可以指定下列属性：

**资源限值**

Kubernetes 可以限制工作负载消耗的资源。​工作负载可以指定在正常操作下预期使用的资源上限。​如果工作负载发生故障或拥有意外负载，则资源限制可防止工作负载消耗过多的资源并影响其他工作负载。​

**资源请求**

工作负载可以声明其所需的最低资源。​Kubernetes 按照工作负载跟踪请求的资源，并且在集群资源不足时阻止部署新的工作负载。​资源请求可确保工作负载获得所需的资源。​

## pod上的资源配额

在创建工作负载的时候，可以给每个pod设置资源配额

以下这个pod，同时设置了request的请求配额与limits的资源限制，以下为单位解读

memory单位： 1Mi == 1024KB  1M == 1000KB 不带单位就是字节

cpu单位： 1 == 1000m 这个可以带小数点，比如1.5 == 1500m

```yaml
cat > quota.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-limit
  labels:
    name: pod-limit
spec:
  containers:
  - name: app
    image: registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:latest
    imagePullPolicy: IfNotPresent
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "100m"
EOF
```

创建并确认配额

```bash
[student@workstation ~]$ oc describe -f quota.yml | grep -A 6 Limits
    Limits:
      cpu:     100m
      memory:  128Mi
    Requests:
      cpu:        100m
      memory:     64Mi
    Environment:  <none>
```

虽然看上去设置了适当的配额，但是这依旧会带来问题，比如一个pod占用100m CPU，那2个呢，3个呢，也就是说虽然给单个pod设置了资源配额，但对方可以通过创建多个来获得更多的资源，此时就需要通过给整个project设置上限来规避此问题

## 项目级别设置资源限额

先创建一个project

```bash
oc new-project lixiaohui
```

给lixiaohui整个project设置配额

```yaml
cat > project-quota.yml <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: lixiaohuiquota
  namespace: lixiaohui
spec:
  hard:
    pods: "1"
    requests.cpu: "1"
    requests.memory: "1Gi"
    limits.cpu: "2"
    limits.memory: "2Gi"
EOF
```

创建并查看

这样比较直观的显示了我们目前可以申请多少资源

```bash
[student@workstation ~]$ oc create -f project-quota.yml
resourcequota/lixiaohuiquota created
[student@workstation ~]$ oc get -f project-quota.yml
NAME             AGE   REQUEST                                                LIMIT
lixiaohuiquota   8s    pods: 0/1, requests.cpu: 0/1, requests.memory: 0/1Gi   limits.cpu: 0/2, limits.memory: 0/2Gi
```

我们试着创建一个超过项目配额的资源请求，看看是否失败

我们申请的内存不超标，但是CPU不管是request还是limit，都是超标的

```yaml
cat > exec-pod.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  namespace: lixiaohui
spec:
  containers:
  - name: app
    image: registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:latest
    imagePullPolicy: IfNotPresent
    resources:
      requests:
        memory: "64Mi"
        cpu: "150m"
      limits:
        memory: "128Mi"
        cpu: "150m"
EOF
```

从下面的报错来看，你超过限额就不行，我们通过给project设置限额，杜绝了用多个pod来绕过限额的可能

```bash
[student@workstation ~]$ oc create -f exec-pod.yml
Error from server (Forbidden): error when creating "exec-pod.yml": pods "frontend" is forbidden: exceeded quota: lixiaohuiquota, requested: limits.cpu=15,requests.cpu=15, used: limits.cpu=0,requests.cpu=0, limited: limits.cpu=2,requests.cpu=1
```

## 项目级别设置数量限额

```yaml
cat > number-quota.yml <<-EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: number-quota
  namespace: lixiaohui  
spec:
  hard:
    count/pods: "1"
EOF
```

创建并验证

```bash
[student@workstation ~]$ oc create -f number-quota.yml
resourcequota/number-quota created
[student@workstation ~]$ oc get -f number-quota.yml
NAME           AGE   REQUEST           LIMIT
number-quota   3s    count/pods: 0/1
```

我们来创建一个3副本的deployment

```yaml
cat > deployment.yml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: lixiaohui    
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "64Mi"
            cpu: "150m"
          limits:
            memory: "128Mi"
            cpu: "150m"
EOF
```

创建并验证发现，只有1个在线

```bash
[student@workstation ~]$ oc create -f deployment.yml
deployment.apps/nginx-deployment created
[student@workstation ~]$ oc get -f deployment.yml
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1/3     1            1           4s
```

查询为什么只有一个在线

清晰的看到，是因为超过了pod数量才失败

```bash
[student@workstation ~]$ oc get replicasets.apps
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-577877fb79   3         1         1       48s
[student@workstation ~]$ oc describe replicasets.apps nginx-deployment-577877fb79
...
Events:
  Type     Reason            Age                From                   Message
  ----     ------            ----               ----                   -------
  Normal   SuccessfulCreate  63s                replicaset-controller  Created pod: nginx-deployment-577877fb79-2lxp6
  Warning  FailedCreate      63s                replicaset-controller  Error creating: pods "nginx-deployment-577877fb79-rtj2j" is forbidden: exceeded quota: lixiaohuiquota, requested: pods=1, used: pods=1, limited: pods=1
```

## 跨project的集群级别配额

集群管理员可能会对资源应用限制，而不限于单个命名空间。例如，一组开发人员管理着许多命名空间。​命名空间配额可以限制每个命名空间的 RAM 使用量。​但是，集群管理员无法限制该开发人员组管理的所有工作负载的总 RAM 使用量。

集群资源配额遵循与命名空间资源配额类似的结构。​但是，<mark>集群资源配额使用选择器来选择应用配额的命名空间</mark>。​

这里我们给具有quota=lixiahui的所有namespace设置了配额

```yaml
cat > cluster-quota.yml <<-EOF
apiVersion: quota.openshift.io/v1
kind: ClusterResourceQuota
metadata:
  name: cluster-quota
spec:
  quota:
    hard:
      limits.memory: 400Mi
  selector:
    annotations: {}
    labels:
      matchLabels:
        quota: lixiahui
EOF
```

创建并确认

```bash
[student@workstation ~]$ oc create -f cluster-quota.yml
clusterresourcequota.quota.openshift.io/cluster-quota created
[student@workstation ~]$ oc get -f cluster-quota.yml
NAME            AGE
cluster-quota   3s
[student@workstation ~]$ oc describe -f cluster-quota.yml
Name:           cluster-quota
Created:        48 seconds ago
Labels:         <none>
Annotations:    <none>
Namespace Selector: []
Label Selector: quota=lixiahui
AnnotationSelector: map[]
Resource        Used    Hard
--------        ----    ----
limits.memory      0       400Mi
```

我们创建一个zhangsan的project，然后给这个project添加上预期的标签，再创建一个名为lixiaohui的pod

```bash
oc new-project zhangsan
oc label namespace zhangsan quota=lixiahui
oc describe project zhangsan
Name:                   zhangsan
Created:                23 seconds ago
Labels:                 kubernetes.io/metadata.name=zhangsan
                        pod-security.kubernetes.io/audit=restricted
                        pod-security.kubernetes.io/audit-version=v1.24
                        pod-security.kubernetes.io/warn=restricted
                        pod-security.kubernetes.io/warn-version=v1.24
                        quota=lixiahui
```

看上去我们预期的quota=lixiahui标签已经存在，先来看看集群配额是否已经选中zhangsan

没问题，已经选中了zhangsan

```bash
[student@workstation ~]$ oc describe clusterresourcequotas.quota.openshift.io cluster-quota
Name:           cluster-quota
Created:        3 minutes ago
Labels:         <none>
Annotations:    <none>
Namespace Selector: ["zhangsan"]
Label Selector: quota=lixiahui
AnnotationSelector: map[]
Resource        Used    Hard
--------        ----    ----
limits.cpu      0       400Mi
```

我们创建一个pod看看

```yaml
cat > lixiaohui-pod.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: lixiaohui
  namespace: zhangsan
spec:
  containers:
  - name: app
    image: registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:latest
    imagePullPolicy: IfNotPresent
    resources:
      requests:
        memory: "64Mi"
        cpu: "150m"
      limits:
        memory: "128Mi"
        cpu: "150m"
EOF
```

查看配额，然后创建出来，再看看配额

```bash
[student@workstation ~]$ oc create -f lixiaohui-pod.yml
pod/lixiaohui created
```

再看的时候，就用了128Mi了

```bash
[student@workstation ~]$ oc describe clusterresourcequotas.quota.openshift.io cluster-quota
Name:           cluster-quota
Created:        19 minutes ago
Labels:         <none>
Annotations:    kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"quota.openshift.io/v1","kind":"ClusterResourceQuota","metadata":{"annotations":{},"name":"cluster-quota"},"spec":{"quota":{"hard":{"limits.memory":"400Mi"}},"selector":{"annotations":{},"labels":{"matchLabels":{"quota":"lixiahui"}}}}}

Namespace Selector: ["zhangsan"]
Label Selector: quota=lixiahui
AnnotationSelector: map[]
Resource        Used    Hard
--------        ----    ----
limits.memory   128Mi   400Mi
```

我们再创建一个lisi的project，并分配quota=lixiaohui的标签，然后再创建一个pod，如果集群资源配额又上升，就说明不管你在哪儿创建，都会受到这个限制

```bash
oc new-project lisi
oc label namespace lisi quota=lixiahui
```

我们创建一个pod看看

```yaml
cat > lixiaohui-pod.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: lixiaohui
  namespace: lisi
spec:
  containers:
  - name: app
    image: registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:latest
    imagePullPolicy: IfNotPresent
    resources:
      requests:
        memory: "64Mi"
        cpu: "150m"
      limits:
        memory: "128Mi"
        cpu: "150m"
EOF
```

创建并验证配额是否上升，从刚才的128，上升到256，说明不管你在哪个project，都会受到限制

```bash
[student@workstation ~]$ oc create -f lixiaohui-pod.yml
[student@workstation ~]$ oc describe clusterresourcequotas.quota.openshift.io cluster-quota
Name:           cluster-quota
Created:        25 minutes ago
Labels:         <none>
Annotations:    kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"quota.openshift.io/v1","kind":"ClusterResourceQuota","metadata":{"annotations":{},"name":"cluster-quota"},"spec":{"quota":{"hard":{"limits.memory":"400Mi"}},"selector":{"annotations":{},"labels":{"matchLabels":{"quota":"lixiahui"}}}}}

Namespace Selector: ["zhangsan" "lisi"]
Label Selector: quota=lixiahui
AnnotationSelector: map[]
Resource        Used    Hard
--------        ----    ----
limits.memory   256Mi   400Mi
```

# 项目级别的LimitRange

