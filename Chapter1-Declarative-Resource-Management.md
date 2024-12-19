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

# 两种管理Kubernetes资源的方法

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

# 管理 Kubernetes 清单

你可能会觉得上面的yaml较为复杂，结构不清晰，这在初学的时候很正常，以下是了解结构的好方法：

## dry-run

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

## YAML的图形化视图

打开Web控制台--工作负载--Pod--创建pod

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter0/webview-yaml.png)

## Explain

`explain` 命令提供清单中任何字段的详细信息。​例如，使用 `oc explain deployment.spec.template.spec` 命令来查看指定部署清单内容器集对象的字段描述。​

```bash
[student@workstation ~]$ oc explain pod
[student@workstation ~]$ oc explain pod.metadata
```

## diff

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
