```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 声明式资源管理

# 课程目标

- 从存储为 YAML 文件的资源清单部署和更新应用。​

- 从 Kustomize 增强的资源清单部署和更新应用。​

# 资源清单

Kubernetes 集群中的应用通常由协同工作的多个资源组成。​每种资源都有一个定义和一个配置，这些定义和配置通常可以有以下两种主流方法实现：

1. #### 命令式

2. #### 声明式

## 使用命令式管理

Kubernetes CLI 使用命令式和声明式命令。​命令式命令执行基于命令的操作，并且使用与操作密切相关的命令名称

例如：以下为命令式创建一个pod

```bash
[student@workstation ~]$ oc login -u admin -p redhatocp https://api.ocp4.example.com:6443
Login successful.

[student@workstation ~]$ oc run nginx --image=nginx
```

使用命令式命令存在一些问题：

- 可再现性受损

- 缺乏版本控制

- 缺乏对 GitOps 的支持

尽管命令时配置存在诸多问题，但依旧是调试的强有力的助手，例如临时在测试环境给deployment设置一下新的镜像、新的环境变量、进入到pod中操作等用于调试目的

## 使用声明式管理

与命令式命令相比，声明式命令是通过使用资源清单来管理资源的首选方式。​<mark>资源清单是 JSON 或 YAML 格式的文件，内含资源定义和配置信息。</mark>​资源清单通过将应用的所有属性封装在一个文件或一组相关文件中，简化了 Kubernetes 资源的管理。​Kubernetes 使用声明式命令来读取资源清单，并将更改应用到集群，以满足资源清单定义的状态。​

资源清单采用 YAML 或 JSON 格式，因此可以进行<mark>版本控制</mark>。​资源清单的版本控制可以跟踪配置更改。​因此，不利的更改可以回滚到较早的版本以支持可恢复性。​

<mark>资源清单可确保精确再现应用，</mark>通常通过一个命令来部署许多资源。​资源清单的可再现性为持续集成和持续开发（CI/CD）的 GitOps 实践的自动化提供了支持。​

举例来说，以下yaml创建了一个名为lixiaohui的pod，而yaml文件本身是源代码，可以放到git中，用于版本控制和再现

```yaml
cat > pod.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: lixiaohuipod
spec:
  containers:
  - name: hello
    image: registry.cn-shanghai.aliyuncs.com/cnlxh/busybox
    imagePullPolicy: IfNotPresent
    command: ['sh', '-c', 'echo "Hello, lixiaohui!" && sleep 3600']
  restartPolicy: OnFailure
EOF
```

```bash
oc apply create/apply -f pod.yml/URL/DIR
```

1. create并不会检查线上资源是否存在，创建资源时如果线上存在同名的资源，会报错
2. apply在创建资源时如果线上存在同名的资源，会用yaml中的参数对线上资源进行更改，如果线上不存在同名资源，则自动执行create操作

## 管理 Kubernetes 清单

你可能会觉得上面的yaml较为复杂，结构不清晰，这在初学的时候很正常，以下是了解结构的好方法：

### dry-run

使用带有 `--dry-run=client/server` 选项的命令式命令生成与命令式命令对应的yaml清单。

以下命令将dry-run设置为client，并使其输出yaml格式，你会发现yaml的基本结构就出现了，save-config把本次命令式的配置保存在注解中

```bash
[student@workstation ~]$ oc create deployment nginx --image=nginx --dry-run=client -o yaml --save-config
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"kind":"Deployment","apiVersion":"apps/v1","metadata":{"name":"nginx","creationTimestamp":null,"labels":{"app":"nginx"}},"spec":{"replicas":1,"selector":{"matchLabels":{"app":"nginx"}},"template":{"metadata":{"creationTimestamp":null,"labels":{"app":"nginx"}},"spec":{"containers":[{"name":"nginx","image":"nginx","resources":{}}]}},"strategy":{}},"status":{}}
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

以下命令将dry-run设置为server，你就可以在不真的创建的前提下，去测试你的命令或yaml服务器是否可以接受并创建

```bash
[student@workstation ~]$ oc run nginx-2 --image=nginx --dry-run=server
pod/nginx-2 created (server dry run)
```

### YAML的图形化视图

打开Web控制台--工作负载--Pod--创建pod

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter0/webview-yaml.png)

### Explain

`explain` 命令提供清单中任何字段的详细信息。​例如，使用 `oc explain deployment.spec.template.spec` 命令来查看指定部署清单内容器集对象的字段描述。​

```bash
[student@workstation ~]$ oc explain pod
[student@workstation ~]$ oc explain pod.metadata
```

### diff

oc diff可以辅助你了解线上资源和现有的yaml文件中，哪里有变化

我们先来创建一个deployment

```yaml
cat > deployment.yml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lixiaohui-deployment
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
        image: nginx
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
EOF
```

```bash
[student@workstation ~]$ oc apply -f deployment.yml
```

完成创建后，我们稍微修改以下deployment.yml文件内容，将镜像改为httpd，之后我们运行oc diff

```bash
[student@workstation ~]$ oc diff -f deployment.yml
...
-  generation: 1
+  generation: 2
...
-      - image: nginx
+      - image: httpd
```

# Kustomize 覆盖

Kustomize 是一个配置管理工具，可以对应用配置和组件进行声明式更改，并保留原始的基础 YAML 文件。​比如说，我们经常要创建`deployment``service`等等资源的时候，其实也不过就是镜像、名称、标签等稍微改改就能用于新的部署，而Kustomize就是做这个的，<mark>我们先写好常用的各类yaml文件用于基本的通用模板，然后用Kustomize来完成新需求中的参数覆盖模板里的参数，就能用现成的模板和新参数来完成新的部署，这样我们有新的需求时，就不用每次都写很长的yaml了</mark>

## Kustomize 文件结构

Kustomize 具有 base和 overlay 的概念，根目录中必须包含 `kustomization.yaml` 文件

### base目录

base就是我们说的通用模板，base目录包含 `kustomization.yaml` 文件。​`kustomization.yaml` 文件具有 list resource 字段，用于包含所有资源文件

下面是base目录的结构：

```textile
base
├── configmap.yaml
├── deployment.yaml
├── secret.yaml
├── service.yaml
├── route.yaml
└── kustomization.yaml
```

base目录中的`kustomization.yaml` 文件内容示例如下：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- configmap.yaml
- deployment.yaml
- secret.yaml
- service.yaml
- route.yaml
```

### base实例

创建出base目录，并在目录中，创建出通用的模板以及kustomization.yaml

```bash
mkdir base
cd base
```

```yaml
cat > Secret.yml <<-EOF
apiVersion: v1
data:
  password: QUJDYWJjMTIz
kind: Secret
metadata:
  name: mysqlpass
EOF
```

```yaml
cat > Deployment.yml <<-EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
        - name: mysqlname
          image: mysql
          imagePullPolicy: IfNotPresent
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysqlpass
                  key: password
EOF
```

```yaml
cat > Service.yml <<-EOF
apiVersion: v1
kind: Service
metadata:
  name: nodeservice
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 80
      nodePort: 31788
EOF
```

最后创建出kustomization.yaml，在这个文件中，我们包含上我们刚创建的3个资源

```yaml
cat > kustomization.yaml <<-EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: lxh-base-kustomization
resources:
- Secret.yml
- Deployment.yml
- Service.yml
EOF
```

查看一下目前base目录的结构

```bash
[student@workstation base]$ pwd
/home/student/base
[student@workstation base]$ tree .
.
├── Deployment.yml
├── kustomization.yaml
├── Secret.yml
└── Service.yml

0 directories, 4 files
```

以上在base目录中的文件将用于生成：

1. 一个名为mysqlpass的机密

2. 一个名为nginx-deployment的部署，此部署的pod将具有app: nginx标签，并引用mysqlpass机密作为密码，密码值为ABCabc123

3. 一个名为nodeservice的服务，监听在8000端口，收到请求后，转发给具有app: nginx标签的pod，并启用了31788的nodePort

至此，我们的三种类型的通用模板就完成了，可以开始overlay的参数覆盖部分了

### 覆盖overlay目录

Kustomize 覆盖声明式 YAML 构件或补丁，会覆盖base设置而不修改base里的原始文件。​覆盖目录包含 `kustomization.yaml` 文件。​`kustomization.yaml` 文件可以将一个或多个目录作为基础目录。​多个覆盖可以使用一个通用的基础 kustomization 目录。​

具体来说，加入overlay之后的文件结构如下：

```bash
[student@workstation ~]$ tree .
base
├── configmap.yaml
├── deployment.yaml
├── secret.yaml
├── service.yaml
├── route.yaml
└── kustomization.yaml
overlay
└── development
    └── kustomization.yaml
└── testing
    └── kustomization.yaml
└── production
    ├── kustomization.yaml
    └── patch.yaml
```

overlay的`kustomization.yaml` 文件中必须写明，对哪类资源做哪些必要的更改

### overlay实例

**创建开发环境**

```bash
cd
mkdir -p overlays/development
cd overlays/development
```

创建开发环境的kustomization.yaml 文件：

Kustomize 功能特性列表参阅：

```text
https://kubernetes.io/zh-cn/docs/tasks/manage-kubernetes-objects/kustomization/#kustomize-feature-list
```

1. patchesStrategicMerge补丁将会更新nginx-deployment这个Deployment
2. patchesJson6902补丁也会更新nginx-deployment这个Deployment
3. overlay的对象是刚创建的base目录下的内容
4. 全体对象添加env: dev
5. 禁止添加hash后缀
6. 产生两个新的configmap和secret

```yaml
cat > kustomization.yaml <<-EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: lxh-dev
patches:
  - path: patchesStrategicMerge-demo.yaml
    target:
      kind: Deployment
      name: nginx-deployment
    options:
      allowNameChange: true
  - path: patchesJson6902-demo.yaml
    target:
      kind: Deployment
      name: nginx-deployment
    options:
      allowNameChange: true
resources:
  - ../../base
commonLabels:
  env: dev
generatorOptions:
  disableNameSuffixHash: true
configMapGenerator:
- name: cmusername
  files:
    - configmap-1.yml
- name: cmage
  literals:
    - cmage=18
secretGenerator:
- name: username
  files:
    - secret-1.yml
  type: Opaque
- name: secrettest
  literals:
    - password=LiXiaoHui
  type: Opaque
EOF
```

#### 生成器Generator

在kustomization.yaml，如果用文件来生成configmap和secret，会将文件名也作为数据的一部分，建议用literals

生成configmap和secret的文件

```text
cat > configmap-1.yml <<-EOF
username=lixiaohui
EOF
```

```bash
cat > secret-1.yml <<-EOF
username=admin
password=secret
EOF
```

#### 策略性合并与JSON补丁

在 Kustomize 中，patchesStrategicMerge 和 patchesJson6902 都用于修改现有的 Kubernetes 资源。

1. patchesStrategicMerge补丁方式使用 YAML 文件来定义，它允许你直接编辑资源的 YAML 结构，就像编辑原始资源文件一样。这种方式直观且易于理解，特别是对于那些熟悉 Kubernetes 资源配置的人来说。

2. patchesJson6902 使用的是 JSON 补丁（JSON Patch）的方式，这是一种更为灵活和强大的补丁应用方式。JSON 补丁遵循 JSON Patch 规范（RFC 6902），允许执行更复杂的操作，如添加、删除、替换、测试等。这种方式使用 JSON 格式定义，可能在处理复杂的修改时更加强大。

**生成策略性合并补丁**

这里的名字一定要和已有的资源的名称一致

更新deployment的replicas为4

```yaml
cat > patchesStrategicMerge-demo.yaml <<-EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 4
EOF
```

**生成JSON补丁**

新增deployment下的pod标签为dev: release1

```json
cat > patchesJson6902-demo.yaml <<-EOF
[
  {
    "op": "add",
    "path": "/spec/template/metadata/labels/dev",
    "value": "release1"
  }
]
EOF
```

目前的文件列表：

```bash
[student@workstation development]$ tree .
.
├── configmap-1.yml
├── kustomization.yaml
├── patchesJson6902-demo.yaml
├── patchesStrategicMerge-demo.yaml
└── secret-1.yml

0 directories, 5 files
```

#### 验证overlay最终成果

```bash
[student@workstation development]$ kubectl kustomize ./
```

输出

```text
apiVersion: v1
data:
  cmage: "18"
kind: ConfigMap
metadata:
  labels:
    env: dev
  name: cmage
  namespace: lxh-dev
---
apiVersion: v1
data:
  configmap-1.yml: |
    username=lixiaohui
kind: ConfigMap
metadata:
  labels:
    env: dev
  name: cmusername
  namespace: lxh-dev
---
apiVersion: v1
data:
  password: QUJDYWJjMTIz
kind: Secret
metadata:
  labels:
    env: dev
  name: mysqlpass
  namespace: lxh-dev
---
apiVersion: v1
data:
  password: TGlYaWFvSHVp
kind: Secret
metadata:
  labels:
    env: dev
  name: secrettest
  namespace: lxh-dev
type: Opaque
---
apiVersion: v1
data:
  secret-1.yml: dXNlcm5hbWU9YWRtaW4KcGFzc3dvcmQ9c2VjcmV0Cg==
kind: Secret
metadata:
  labels:
    env: dev
  name: username
  namespace: lxh-dev
type: Opaque
---
apiVersion: v1
kind: Service
metadata:
  labels:
    env: dev
  name: nodeservice
  namespace: lxh-dev
spec:
  ports:
  - nodePort: 31788
    port: 8000
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
    env: dev
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
    env: dev
  name: nginx-deployment
  namespace: lxh-dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      env: dev
  template:
    metadata:
      labels:
        app: new-label
        env: dev
    spec:
      containers:
      - env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              key: password
              name: mysqlpass
        image: mysql
        imagePullPolicy: IfNotPresent
        name: mysqlname
```

#### 发布开发环境

```bash
kubectl create namespace lxh-dev
kubectl apply -k .
```

```text
configmap/cmage created
configmap/cmusername created
secret/mysqlpass created
secret/secrettest created
secret/username created
service/nodeservice created
deployment.apps/nginx-deployment created
```

查询创建的内容

发现我们新configmap和secret已经生效，两个补丁也都生效了，一个补丁将deployment的pod数量该为4，一个补丁添加了dev=release1的标签

```bash
[student@workstation development]$ kubectl get configmaps -n lxh-dev
NAME               DATA   AGE
cmage              1      41s
cmusername         1      41s
kube-root-ca.crt   1      11m
[student@workstation development]$ kubectl get secrets -n lxh-dev
NAME         TYPE     DATA   AGE
mysqlpass    Opaque   1      47s
secrettest   Opaque   1      47s
username     Opaque   1      47s

[student@workstation development]$ kubectl get service -n lxh-dev
NAME          TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
nodeservice   NodePort   10.106.128.145   <none>        8000:31788/TCP   51s

[student@workstation development]$ kubectl get deployments.apps -n lxh-dev
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   4/4     4            4           55s

[student@workstation development]$ kubectl get pod --show-labels  -n lxh-dev
NAME                                READY   STATUS    RESTARTS   AGE   LABELS
nginx-deployment-6f86fd678b-bv688   1/1     Running   0          64s   app=nginx,dev=release1,env=dev,pod-template-hash=6f86fd678b
nginx-deployment-6f86fd678b-wpk49   1/1     Running   0          64s   app=nginx,dev=release1,env=dev,pod-template-hash=6f86fd678b
nginx-deployment-6f86fd678b-wr94h   1/1     Running   0          64s   app=nginx,dev=release1,env=dev,pod-template-hash=6f86fd678b
nginx-deployment-6f86fd678b-xxkbw   1/1     Running   0          64s   app=nginx,dev=release1,env=dev,pod-template-hash=6f86fd678b
```
