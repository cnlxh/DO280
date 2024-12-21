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

### LimitRange的介绍

在 Kubernetes 中，`LimitRange` 是一种关键的资源类型，用于管理和限制命名空间中的资源使用。通过设置资源的默认请求和限制，`LimitRange` 确保了集群资源的合理分配和高效利用，避免了资源的浪费和过度使用。以下是 `LimitRange` 的一些必要性及其示例说明：

#### 1. 防止资源滥用

在多租户环境中，不同团队或应用共享同一个 Kubernetes 集群。如果没有资源限制，某些应用可能会占用过多资源，导致其他应用的性能下降，甚至无法正常运行。`LimitRange` 通过设置最大和最小资源限制，防止单个容器或 Pod 占用过多资源，从而确保各个应用能够公平地共享集群资源。

**示例**：假设一个命名空间中运行了多个应用，如果没有 `LimitRange` 限制，一个内存密集型的应用可能会占用大部分内存资源，导致其他应用因资源不足而崩溃。通过设置 `LimitRange`，可以限制每个应用的最大内存使用量，保证所有应用都能获得足够的资源运行。

#### 2. 提供合理的默认值

有些用户在定义 Pod 时可能没有指定资源请求和限制。这样一来，Kubernetes 调度器在分配资源时可能会过度或不足分配，影响集群的性能和稳定性。通过设置 `LimitRange`，可以为资源请求和限制提供合理的默认值，即使用户未显式指定，Kubernetes 也能根据这些默认值进行调度。

**示例**：某个命名空间中的 Pod 没有定义资源请求和限制，导致调度器无法合理分配资源。通过设置 `LimitRange`，可以为 Pod 设置默认的资源请求（如 256Mi 内存）和限制（如 512Mi 内存），确保调度器能够合理分配资源，保证集群的稳定运行。

<mark>具体来说，LimitRange可以防止资源被不合理的分配或Pod过多的占用资源，LimitRange也会给Pod提供默认的request和limit</mark>

## LimitRange 案例

我们来做个测试，创建一个deployment，并观察其默认是否有resources的字段，然后设置limitrange，再推出新的pod，继续观察，不过需要注意的是，LimitRange不影响现有的 pod。​

```yaml
cat > deployment.yml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
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
EOF
```

创建并观察pod是否会被自动添加resources资源限制，结果是pod跟随deployment要求，并没有添加限制资源

```bash
oc create -f deployment.yml
```

看看默认的resources字段，根本就没有这个字段，说明并没有添加限额

```bash
[student@workstation ~]$ oc get pod
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6645d8bb58-5hbzp   1/1     Running   0          119s
[student@workstation ~]$ oc describe pod nginx-deployment-6645d8bb58-5hbzp
```

我们来创建一个LimitRange

```yaml
cat > limitrange.yml <<-EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: lixiaohui-limit
spec:
  limits:
  - default:
      cpu: 500m            # 默认 CPU 限制为 500 毫核
      memory: 512Mi        # 默认内存限制为 512 Mi
    defaultRequest:
      cpu: 250m            # 默认 CPU 请求为 250 毫核
      memory: 256Mi        # 默认内存请求为 256 Mi
    max:
      cpu: "1"             # 最大 CPU 限制为 1 核
      memory: 1Gi          # 最大内存限制为 1 Gi
    min:
      cpu: 125m            # 最小 CPU 请求为 125 毫核
      memory: 128Mi        # 最小内存请求为 128 Mi
    type: Container        # 适用于容器级别
EOF
```

`min` 和 `max` 的作用

- `min`：指定容器或 Pod 可以请求的最小资源量。这个值确保每个容器或 Pod 至少请求一定量的资源，以避免资源不足的问题。

- `max`：指定容器或 Pod 可以请求的最大资源量。这个值限制了容器或 Pod 不能请求过多的资源，以防止单个容器或 Pod 占用过多的资源。

`default` 和 `defaultRequest` 的作用

- `default`：为没有显式指定资源限制的容器或 Pod 设置默认的资源限制。

- `defaultRequest`：为没有显式指定资源请求的容器或 Pod 设置默认的资源请求。

```bash
oc create -f limitrange.yml
```

创建后，发现pod依然没有resources，那是因为并不影响已有的pod，所以我们删除这个pod，由deployment自动产生一个新的，再describe看看

```bash
[student@workstation ~]$ oc delete pod nginx-deployment-6645d8bb58-5hbzp
pod "nginx-deployment-6645d8bb58-5hbzp" deleted

[student@workstation ~]$ oc get pod
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6645d8bb58-qt42t   1/1     Running   0          5s

[student@workstation ~]$ oc describe pod nginx-deployment-6645d8bb58-qt42t
...
    Limits:
      cpu:     500m
      memory:  512Mi
    Requests:
      cpu:        250m
      memory:     256Mi
```

我们发现虽然deployment本身没有提供限额，但是pod却根据limitrange添加了限额，这样就可以防止资源被过多浪费，不过需要注意的是，limitrange并没有修改deployment，而是直接影响pod，如果需要，我们可以用下面的方式修改deployment的限额

```bash
[student@workstation ~]$ oc set resources deployment nginx-deployment --requests cpu=100m,memory=200Mi --limits cpu=150m,memory=300Mi
deployment.apps/nginx-deployment resource requirements updated
```

CPU 或内存密钥的值必须遵循下列规则：

- `max` 值必须大于或等于 `default` 值。​

- `default` 值必须大于或等于 `defaultRequest` 值。​

- `defaultRequest` 值必须大于或等于 `min` 值。

# 项目模板和 self-provisioner 角色

## 项目模板

### Kubernetes Namespace 的问题

在 Kubernetes 中，Namespace 提供了一种逻辑隔离的方式，用于将集群中的资源分组。然而，Namespace 有一些局限性和挑战：

1. **权限管理的复杂性**：
   
   - 在 Kubernetes 中，对 Namespace 进行细粒度的权限管理可能会变得复杂。RBAC（角色绑定访问控制）配置需要手动操作，容易出错和管理困难。

2. **资源配额控制的局限**：
   
   - 虽然 Kubernetes 支持通过 ResourceQuota 来限制 Namespace 的资源使用，但缺乏内置的工具来监控和自动调整这些配额。管理多个 Namespace 的资源配额可能会变得繁琐。

3. **缺乏项目级别的隔离和管理**：
   
   - Namespace 主要关注资源隔离，但缺乏更高级别的项目管理功能。例如，无法通过 Namespace 实现环境（开发、测试、生产）的自动化和版本控制。

4. **不支持多租户管理**：
   
   - Namespace 在处理多租户环境时存在挑战，无法为每个租户提供完全隔离的环境。共享集群资源可能导致安全和性能问题。

### OpenShift Project 的优势

OpenShift 项目（Project）基于 Kubernetes Namespace，但它引入了一些额外的功能来解决上述问题。OpenShift Project 提供了更高级别的资源管理和控制功能，具体包括：

1. **增强的权限管理**：
   
   - OpenShift 提供了更简化和直观的权限管理界面，通过内置的角色和绑定机制，管理员可以更轻松地管理用户和组的访问权限。

2. **自动化资源配额管理**：
   
   - OpenShift Project 允许管理员为每个项目设置资源配额，并提供工具监控和调整这些配额，确保资源分配合理且高效。

3. **项目级别的管理功能**：
   
   - OpenShift 项目允许管理员将应用、构建、部署、服务和配置资源组织在一起，提供了更高级别的管理视图。这有助于对不同环境（如开发、测试和生产）进行更好的管理和控制。

4. **多租户支持**：
   
   - OpenShift 提供了强大的多租户管理功能，每个项目可以作为一个独立的租户环境，确保资源、网络和安全的完全隔离。这为不同团队和应用提供了更安全的运行环境。

### 项目概述

OpenShift 引入了一些项目，以改进命名空间的安全性和用户使用体验。​OpenShift API 服务器增加了 `Project` 资源类型。​当您发出查询以列出项目时，API 服务器会列出命名空间，将可见命名空间过滤到您的用户，并以项目格式返回可见命名空间。​

此外，OpenShift 还引入了 `ProjectRequest` 资源类型。​在您创建项目请求时，OpenShift API 服务器从模板创建命名空间。​通过使用模板，集群管理员可以自定义命名空间创建。​例如，集群管理员可以确保新的命名空间具有特定的权限、​资源配额或限值范围。​

这些功能提供命名空间的自助服务管理。​集群管理员可以允许用户创建命名空间，而不必允许用户修改命名空间元数据。​管理员还可以自定义命名空间的创建，以确保命名空间遵循组织要求。​

### 规划项目模板

您可以将任何命名空间资源添加到项目模板。​例如，您可以添加以下类型的资源：

**角色和角色绑定**

添加角色和角色绑定到模板，以在新项目中授予特定权限。​默认模板会向请求项目的用户授予 `admin` 角色。​您可以保留此权限或使用其他类似权限，例如将 `admin` 角色授予一组用户。​您还可以添加不同的权限，例如对特定资源类型更精细的权限。​

**资源配额和限值范围**

将资源配额添加到项目模板，以确保所有新项目都有资源限制。​如果您添加资源配额，则创建工作负载需要明确的资源限值声明。​考虑添加限值范围，以减少创建工作负载的工作量。​

即使在所有命名空间中都有配额，用户也可以创建项目来继续向集群添加工作负载。​如果存在这种情况，请考虑向集群添加集群资源配额。​

**网络政策**

向模板添加网络政策，以实施组织网络隔离要求。​

### 创建项目模板

`oc adm create-bootstrap-project-template` 命令输出一个模板，您可以使用它来创建自己的项目模板。​

此模板具有与 OpenShift 中默认项目创建相同的行为。​该模板添加了一个角色绑定，它将新命名空间上的 `admin` 集群角色授予请求项目的用户。​

从模板可以看到，我们在创建project的时候，可以直接分配管理员等RBAC信息，我们也可以根据需要添加配额等信息

```yaml
[student@workstation ~]$ oc adm create-bootstrap-project-template -o yaml > template.yml
[student@workstation ~]$ cat template.yml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  creationTimestamp: null
  name: project-request
objects:
- apiVersion: project.openshift.io/v1
  kind: Project
  metadata:
    annotations:
      openshift.io/description: ${PROJECT_DESCRIPTION}
      openshift.io/display-name: ${PROJECT_DISPLAYNAME}
      openshift.io/requester: ${PROJECT_REQUESTING_USER}
    creationTimestamp: null
    name: ${PROJECT_NAME}
  spec: {}
  status: {}
- apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    creationTimestamp: null
    name: admin
    namespace: ${PROJECT_NAME}
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: admin
  subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: ${PROJECT_ADMIN_USER}
parameters:
- name: PROJECT_NAME
- name: PROJECT_DISPLAYNAME
- name: PROJECT_DESCRIPTION
- name: PROJECT_ADMIN_USER
- name: PROJECT_REQUESTING_USER
```

我们没有看到配额信息，只有rbac，我们来手工添加配额等信息，手工写配额信息会出错，所以先创建好之后，转成yaml，并复制到项目模板

```yaml
[student@workstation ~]$ oc get limitranges lixiaohui-limit -o yaml
apiVersion: v1
kind: LimitRange
metadata:
  creationTimestamp: "2024-12-21T11:18:15Z"
  name: lixiaohui-limit
  namespace: default
  resourceVersion: "99495"
  uid: b275749e-eb52-47c2-a7ea-5fe5acd81157
spec:
  limits:
  - default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 250m
      memory: 256Mi
    max:
      cpu: "1"
      memory: 1Gi
    min:
      cpu: 125m
      memory: 128Mi
    type: Container
```

手工添加到参数上面就行

```yaml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  creationTimestamp: null
  name: project-request
objects:
- apiVersion: project.openshift.io/v1
  kind: Project
  metadata:
    annotations:
      openshift.io/description: ${PROJECT_DESCRIPTION}
      openshift.io/display-name: ${PROJECT_DISPLAYNAME}
      openshift.io/requester: ${PROJECT_REQUESTING_USER}
    creationTimestamp: null
    name: ${PROJECT_NAME}
  spec: {}
  status: {}
- apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    creationTimestamp: null
    name: admin
    namespace: ${PROJECT_NAME}
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: admin
  subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: ${PROJECT_ADMIN_USER}
- apiVersion: v1
  kind: LimitRange
  metadata:
    name: lixiaohui-limit
    namespace: ${PROJECT_NAME} # 这里手工写了变量
  spec:
    limits:
    - default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 250m
        memory: 256Mi
      max:
        cpu: "1"
        memory: 1Gi
      min:
        cpu: 125m
        memory: 128Mi
      type: Container
parameters:
- name: PROJECT_NAME
- name: PROJECT_DISPLAYNAME
- name: PROJECT_DESCRIPTION
- name: PROJECT_ADMIN_USER
- name: PROJECT_REQUESTING_USER
```

改到满意之后，用下面的方法来完成创建

```bash
[student@workstation ~]$ oc create -f template.yml -n openshift-config
template.template.openshift.io/project-request created

[student@workstation ~]$ oc get template -n openshift-config
NAME              DESCRIPTION   PARAMETERS    OBJECTS
project-request                 5 (5 blank)   3
```

我们来更新一下集群资源，让集群在创建新的project的时候，用我们的模板

```bash
[student@workstation ~]$ oc edit projects.config.openshift.io cluster
...
spec: # 下面是添加的内容
  projectRequestTemplate:
    name: project-request
```

这个一旦更新，我们的apiserver会重新生成，可能没那么快，慢慢等待

```bash
[student@workstation ~]$ oc get co
openshift-apiserver                        4.14.0    False       False         False      32s     APIServerDeploymentAvailable: no apiserver.openshift-apiserver pods available on any node....
```

等上面的更新完成后，我们创建一个project，然后进去创建一个deployment，看看是否会自带我们的限额

```bash
[student@workstation ~]$ oc new-project hello
```

创建一个deployment

```yaml
cat > deployment.yml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
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
EOF
```

发现已经自带了我们的限额参数

```bash
[student@workstation ~]$ oc create -f deployment.yml
[student@workstation ~]$ oc get pod
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6645d8bb58-4jbct   1/1     Running   0          8s
[student@workstation ~]$ oc describe pod nginx-deployment-6645d8bb58-4jbct
...
    Limits:
      cpu:     500m
      memory:  512Mi
    Requests:
      cpu:        250m
      memory:     256Mi
```

### 恢复默认模板

如果需要恢复默认的模板，就清楚spec下面的参数

```bash
[student@workstation ~]$ oc edit projects.config.openshift.io cluster
...
spec: {}
```

如果需要，你可以重新创建project、deployment，验证pod不再自带限额信息

## 管理自行置备权限

具有 `self-provisioner` 集群角色的用户可以创建项目。​默认情况下， self-provisioner 角色绑定到所有经过身份验证的用户。​

```bash
[student@workstation ~]$ oc describe clusterrolebinding.rbac self-provisioners
Name:         self-provisioners
Labels:       <none>
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
Role:
  Kind:  ClusterRole
  Name:  self-provisioner
Subjects:
  Kind   Name                        Namespace
  ----   ----                        ---------
  Group  system:authenticated:oauth
```

如果需要删除经过身份验证的用户创建project的权限，那就删除这个集群角色绑定或者edit一下，清空subjects下面的内容就行，或者你换成别的组或用户，仅授权而已

```bash
[student@workstation ~]$ oc delete clusterrolebindings self-provisioners
clusterrolebinding.rbac.authorization.k8s.io "self-provisioners" deleted
[student@workstation ~]$ oc login -u developer -p developer
Login successful.

You don't have any projects. Contact your system administrator to request a project.
```

我们发现，他已经需要联系管理员才能创建project了

不过我们要注意`Annotations:  rbac.authorization.kubernetes.io/autoupdate: true`这个参数，这个参数确保我们修改的参数不会影响集群工作，也就是说，一旦执行下面的命令，这个clusterrolebind就会重新恢复，也就是说，仅删除到 API 服务器重启那一刻

```bash
[student@workstation ~]$ oc rollout restart deployment -n openshift-apiserver
```

如果你真想永久删除，你得这么做

先禁用自动更新，然后再删除绑定

```bash
oc annotate clusterrolebinding/self-provisioners --overwrite rbac.authorization.kubernetes.io/autoupdate=false
```

```bash
[student@workstation ~]$ oc delete clusterrolebindings self-provisioners
```


