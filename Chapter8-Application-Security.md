```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 应用安全性

# 课程目标

- 创建服务帐户和应用权限，并管理安全上下文约束。​

- 运行需要访问应用集群的 Kubernetes API 的应用。​

- 利用 Kubernetes cron 作业自动执行常规集群和应用管理任务。

# 使用安全性上下文约束来控制应用权限

红帽 OpenShift 提供​*安全上下文约束（SCC）*，这种安全机制限制从 OpenShift 中运行的容器集到主机环境的访问。​SCC 控制以下主机资源：

- 运行特权容器

- 请求容器的额外功能

- 将主机目录用作卷

- 更改容器的 SELinux 上下文

- 更改用户 ID

## 列出scc

集群管理员可以运行以下命令，以列出 OpenShift 定义的 SCC：

```bash
[student@workstation ~]$ oc get scc
NAME                              PRIV    CAPS                   SELINUX     RUNASUSER          FSGROUP     SUPGROUP    PRIORITY     READONLYROOTFS   VOLUMES
anyuid                            false   <no value>             MustRunAs   RunAsAny           RunAsAny    RunAsAny    10           false            ["configMap","csi","downwardAPI","emptyDir","ephemeral","persistentVolumeClaim","projected","secret"]
hostaccess                        false   <no value>             MustRunAs   MustRunAsRange     MustRunAs   RunAsAny    <no value>   false            ["configMap","csi","downwardAPI","emptyDir","ephemeral","hostPath","persistentVolumeClaim","projected","secret"]
hostmount-anyuid                  false   <no value>             MustRunAs   RunAsAny           RunAsAny    RunAsAny    <no value>   false            ["configMap","csi","downwardAPI","emptyDir","ephemeral","hostPath","nfs","persistentVolumeClaim","projected","secret"]
hostnetwork                       false   <no value>             MustRunAs   MustRunAsRange     MustRunAs   MustRunAs   <no value>   false            ["configMap","csi","downwardAPI","emptyDir","ephemeral","persistentVolumeClaim","projected","secret"]
hostnetwork-v2                    false   ["NET_BIND_SERVICE"]   MustRunAs   MustRunAsRange     MustRunAs   MustRunAs   <no value>   false            ["configMap","csi","downwardAPI","emptyDir","ephemeral","persistentVolumeClaim","projected","secret"]
lvms-topolvm-node                 true    <no value>             RunAsAny    RunAsAny           RunAsAny    RunAsAny    <no value>   false            ["configMap","emptyDir","hostPath","secret"]
lvms-vgmanager                    true    <no value>             MustRunAs   RunAsAny           MustRunAs   RunAsAny    <no value>   false            ["configMap","emptyDir","hostPath","secret"]
machine-api-termination-handler   false   <no value>             MustRunAs   RunAsAny           MustRunAs   MustRunAs   <no value>   false            ["downwardAPI","hostPath"]
node-exporter                     true    <no value>             RunAsAny    RunAsAny           RunAsAny    RunAsAny    <no value>   false            ["*"]
nonroot                           false   <no value>             MustRunAs   MustRunAsNonRoot   RunAsAny    RunAsAny    <no value>   false            ["configMap","csi","downwardAPI","emptyDir","ephemeral","persistentVolumeClaim","projected","secret"]
nonroot-v2                        false   ["NET_BIND_SERVICE"]   MustRunAs   MustRunAsNonRoot   RunAsAny    RunAsAny    <no value>   false            ["configMap","csi","downwardAPI","emptyDir","ephemeral","persistentVolumeClaim","projected","secret"]
privileged                        true    ["*"]                  RunAsAny    RunAsAny           RunAsAny    RunAsAny    <no value>   false            ["*"]
restricted                        false   <no value>             MustRunAs   MustRunAsRange     MustRunAs   RunAsAny    <no value>   false            ["configMap","csi","downwardAPI","emptyDir","ephemeral","persistentVolumeClaim","projected","secret"]
restricted-v2                     false   ["NET_BIND_SERVICE"]   MustRunAs   MustRunAsRange     MustRunAs   RunAsAny    <no value>   false            ["configMap","csi","downwardAPI","emptyDir","ephemeral","persistentVolumeClaim","projected","secret"]
```

看来默认情况下，openshift提供以下的scc

```bash
[student@workstation ~]$ oc get scc -o name
securitycontextconstraints.security.openshift.io/anyuid
securitycontextconstraints.security.openshift.io/hostaccess
securitycontextconstraints.security.openshift.io/hostmount-anyuid
securitycontextconstraints.security.openshift.io/hostnetwork
securitycontextconstraints.security.openshift.io/hostnetwork-v2
securitycontextconstraints.security.openshift.io/lvms-topolvm-node
securitycontextconstraints.security.openshift.io/lvms-vgmanager
securitycontextconstraints.security.openshift.io/machine-api-termination-handler
securitycontextconstraints.security.openshift.io/node-exporter
securitycontextconstraints.security.openshift.io/nonroot
securitycontextconstraints.security.openshift.io/nonroot-v2
securitycontextconstraints.security.openshift.io/privileged
securitycontextconstraints.security.openshift.io/restricted
securitycontextconstraints.security.openshift.io/restricted-v2
```

## 查询scc属性

```bash
[student@workstation ~]$ oc describe scc anyuid

Name:                                           anyuid   # SCC（安全上下文约束）的名称
Priority:                                       10       # SCC 的优先级，优先级越高的 SCC 优先匹配
Access:
  Users:                                        <none>   # 没有直接指定用户
  Groups:                                       system:cluster-admins   # 允许使用此 SCC 的组
Settings:
  Allow Privileged:                             false    # 不允许特权容器
  Allow Privilege Escalation:                   true     # 允许提升特权
  Default Add Capabilities:                     <none>   # 默认添加的特权能力（无）
  Required Drop Capabilities:                   MKNOD    # 必须去掉的特权能力
  Allowed Capabilities:                         <none>   # 允许的特权能力（无）
  Allowed Seccomp Profiles:                     <none>   # 允许的 seccomp 配置文件（无）
  Allowed Volume Types:                         configMap,csi,downwardAPI,emptyDir,ephemeral,persistentVolumeClaim,projected,secret   # 允许的卷类型
  Allowed Flexvolumes:                          <all>    # 允许所有类型的 Flexvolume
  Allowed Unsafe Sysctls:                       <none>   # 允许的不安全 sysctls（无）
  Forbidden Sysctls:                            <none>   # 禁止的 sysctls（无）
  Allow Host Network:                           false    # 不允许主机网络
  Allow Host Ports:                             false    # 不允许主机端口
  Allow Host PID:                               false    # 不允许主机 PID 空间
  Allow Host IPC:                               false    # 不允许主机 IPC 空间
  Read Only Root Filesystem:                    false    # 根文件系统不为只读
  Run As User Strategy: RunAsAny
    UID:                                        <none>   # 没有特定的 UID
    UID Range Min:                              <none>   # 没有特定的最小 UID 范围
    UID Range Max:                              <none>   # 没有特定的最大 UID 范围
  SELinux Context Strategy: MustRunAs
    User:                                       <none>   # 没有特定的 SELinux 用户
    Role:                                       <none>   # 没有特定的 SELinux 角色
    Type:                                       <none>   # 没有特定的 SELinux 类型
    Level:                                      <none>   # 没有特定的 SELinux 等级
  FSGroup Strategy: RunAsAny
    Ranges:                                     <none>   # 没有特定的文件系统组范围
  Supplemental Groups Strategy: RunAsAny
    Ranges:                                     <none>   # 没有特定的补充组范围
```

大部分的pod都用的是restricted，因为这个提供 OpenShift 外部资源的有限访问权限

```bash
[student@workstation ~]$ oc get pod -n openshift-console
NAME                         READY   STATUS    RESTARTS   AGE
console-5fb5c88b98-wb7c7     1/1     Running   5          332d
downloads-7984574494-w8d6g   1/1     Running   7          333d
[student@workstation ~]$ oc describe pod -n openshift-console console-5fb5c88b98-wb7c7 | grep scc
                      openshift.io/scc: restricted-v2
```

## 授予scc特权案例

涉及到特权管理，我们就以lab来完成学习

```bash
[student@workstation ~]$ lab start appsec-scc
SUCCESS Waiting for cluster
SUCCESS Remove appsec-scc project
```

用普通账号登录，然后创建一个应用，观察其失败的原因，并分配合适的scc给到serviceaccount，将具有scc的sa绑定到应用，即可解决失败的问题

```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443
```

```bash
oc new-project appsec-scc
```

```bash
oc new-app --name gitlab \
  --image registry.ocp4.example.com:8443/redhattraining/gitlab-ce:8.4.3-ce.0
```

创建一段时间后，来获取一些pod状态发现是失败的

```bash
[student@workstation ~]$ oc get pod
NAME                      READY   STATUS   RESTARTS     AGE
gitlab-6fd4f89dbc-hkr54   0/1     Error    1 (9s ago)   17s
```

获取日志看看，发现对特定的目录没有权限

```bash
[student@workstation ~]$ oc logs gitlab-6fd4f89dbc-hkr54
...
[2024-12-22T08:35:55+00:00] ERROR: directory[/etc/gitlab] (gitlab::default line 26) had an error: Chef::Exceptions::InsufficientPermissions: Cannot create directory[/etc/gitlab] at /etc/gitlab due to insufficient permissions
[2024-12-22T08:35:55+00:00] FATAL: Chef::Exceptions::ChildConvergeError: Chef run process exited unsuccessfully (exit code 1)
```

既然权限不足，那就让应用root权限运行就可以了，我们来创建一个serviceaccount，然后给这个serviceaccount绑定anyuid的scc，让它可以以任何人的uid运行

用管理员登录，给其准备好资源

```bash
[student@workstation ~]$ oc login -u admin -p redhatocp https://api.ocp4.example.com:6443
[student@workstation ~]$ oc create sa gitlab-sa
[student@workstation ~]$ oc adm policy add-scc-to-user anyuid -z gitlab-sa
```

再登录普通用户，将这个具有特权的serviceaccount分配到他自己的资源

```bash
[student@workstation ~]$ oc login -u developer -p developer
[student@workstation ~]$ oc set serviceaccount deployment/gitlab gitlab-sa
[student@workstation ~]$ oc rollout restart deployment gitlab
```

观察一下新的pod是否成功运行

```bash
[student@workstation ~]$ oc get pod
NAME                      READY   STATUS        RESTARTS   AGE
gitlab-7fd875b946-cvq4c   1/1     Running       0          24s
```

ok，看上去没什么问题，我们来访问一下看看

```bash
[student@workstation ~]$ oc expose service/gitlab --port 80 --hostname gitlab.apps.ocp4.example.com
```

访问成功，说明应用成功启动并提供服务了

```bash
[student@workstation ~]$ curl -sL http://gitlab.apps.ocp4.example.com | grep -i 'sign in'
<meta content='Sign in' property='og:title'>
<meta content='Sign in' property='twitter:title'>
<title>Sign in · GitLab</title>
<h3>Existing user? Sign in</h3>
<input type="submit" name="commit" value="Sign in" class="btn btn-save" />
```

# 允许应用访问 Kubernetes API

## 访问API的场景

以下情况下，pod中的应用必须能直接对K8S的API发起请求：

**自动扩展（Auto-scaling）**

- **Kubernetes Horizontal Pod Autoscaler (HPA)**：HPA 会根据 Pod 的 CPU 使用率或其他自定义指标，自动扩展或缩减 Pod 的副本数。HPA 需要访问 Kubernetes API 来监控指标并调整资源。

**监控和日志收集**

- **Prometheus Operator**：Prometheus 监控系统需要访问 Kubernetes API 获取节点、Pod 和服务的指标信息，从而进行资源监控和告警。

- **Fluentd**：日志收集工具如 Fluentd 会从各个 Pod 中收集日志，并将其传输到集中式日志存储。它需要通过 Kubernetes API 获取 Pod 的日志和元数据。

## 使用服务帐户进行应用授权

服务帐户是项目中的 Kubernetes 对象。​服务帐户表示在 pod 中运行的应用的身份。​

要向应用授予对 Kubernetes API 的访问权限，请执行以下操作：

- 创建应用服务帐户。​

- 授予服务帐户对 Kubernetes API 的访问权限。​

- 将服务帐户分配到应用 pod。​

如果 pod 定义不指定服务帐户，则 pod 将会使用 `default` 服务帐户。​OpenShift 不会向 `default` 服务帐户授予任何权限

## 授权案例

我们来创建以下资源：

1. 服务账号

2. 角色/集群角色

3. 角色绑定

最后将服务账号分配到应用，用命令测试权限是否足够

### 创建服务账号

创建一个名为lxh-serviceaccount的账号

```bash
[student@workstation ~]$ oc create serviceaccount lxh-serviceaccount
```

### 创建角色

需要注意的是，角色和集群角色是不同的，角色是本项目内适用，而集群角色是横跨所有项目的

我们这次就来创建一个本项目内的，我们希望这个角色能带来查看pod和deployment的权限，本次创建的角色名为`lxh-role`

```bash
[student@workstation ~]$ oc create role lxh-role --verb=get --resource=deployments,pods
role.rbac.authorization.k8s.io/lxh-role created
[student@workstation ~]$ oc describe role lxh-role
Name:         lxh-role
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources         Non-Resource URLs  Resource Names  Verbs
  ---------         -----------------  --------------  -----
  pods              []                 []              [get]
  deployments.apps  []                 []              [get]
```

### 角色绑定

和角色一样，角色绑定也分为集群角色绑定和角色绑定，我们本次绑定的是本项目内的角色，所以选择普通的绑定即可

```bash
[student@workstation ~]$ oc create rolebinding lxh-bind --role lxh-role --serviceaccount default:lxh-serviceaccount
[student@workstation ~]$ oc describe rolebinding lxh-bind
Name:         lxh-bind
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  Role
  Name:  lxh-role
Subjects:
  Kind            Name                Namespace
  ----            ----                ---------
  ServiceAccount  lxh-serviceaccount  default
```

### 测试权限

经过测试，权限符合预期

```bash
[student@workstation ~]$ oc auth can-i get pod --as system:serviceaccount:default:lxh-serviceaccount
yes
[student@workstation ~]$ oc auth can-i get deployment --as system:serviceaccount:default:lxh-serviceaccount
yes
[student@workstation ~]$ oc auth can-i get configmap --as system:serviceaccount:default:lxh-serviceaccount
no
```

### 现有角色绑定

集群中默认也提供了一些角色，可以用以下方法绑定，就不用创建角色了

```bash
[student@workstation ~]$ oc get clusterrole
...
admin
cluster-admin
edit
view
...
```

有很多角色，可以用以下方法来查询其权限

```bash
[student@workstation ~]$ oc describe clusterrole view
Name:         view
Labels:       kubernetes.io/bootstrapping=rbac-defaults
              rbac.authorization.k8s.io/aggregate-to-edit=true
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources                                             Non-Resource URLs  Resource Names                                          Verbs
  ---------                                             -----------------  --------------                                          -----
  appliedclusterresourcequotas                          []                 []                                                      [get list watch]
  bindings                                              []                 []                                                      [get list watch]
  buildconfigs/webhooks                                 []                 []                                                      [get list watch]
  buildconfigs                                          []                 []                                                      [get list watch]
  buildlogs                                             []                 []                                                      [get list watch]
  builds/log                                            []                 []                                                      [get list watch]
  builds                                                []                 []                                                      [get list watch]
  configmaps                                            []                 []                                                      [get list watch]
```

绑定方法可以用:

```bash
[student@workstation ~]$ oc adm policy add-role-to-user admin system:serviceaccount:default:lxh-serviceaccount --rolebinding-name lxh-bing-2 -n default
clusterrole.rbac.authorization.k8s.io/admin added: "system:serviceaccount:default:lxh-serviceaccount"
```

再测试权限，我们已经可以做更多事了

```bash
[student@workstation ~]$ oc auth can-i get configmap --as system:serviceaccount:default:lxh-serviceaccount
yes
[student@workstation ~]$ oc auth can-i create configmap --as system:serviceaccount:default:lxh-serviceaccount
yes
```

# 通过 Kubernetes Cron 作业维护集群和节点

集群管理员可以使用调度任务来自动执行集群中的维护任务。​其他用户可以创建调度任务以进行常规应用维护。​

**作业**

Kubernetes 作业指定执行一次的任务。​

**Cron 作业**

Kubernetes cron 作业具有定期执行任务的调度。​

当 cron 作业执行到期时，Kubernetes 会创建作业资源。​Kubernetes 从 cron 作业定义中的模板创建这些作业。

## Job

创建一个Job，这个Job只有一个任务，完成后即可退出，就是hello lixiaohui的字符串输出

```yaml
cat > job.yml <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-lixiaohui-job
spec:
  template:
    spec:
      containers:
      - name: pi
        image: registry.ocp4.example.com:8443/openshift/origin-cli:4.12
        imagePullPolicy: IfNotPresent
        command: ["sh",  "-c", "echo hello lixiaohui"]
      restartPolicy: Never
  backoffLimit: 4
EOF
```

看一下任务运行的结果

```bash
[student@workstation ~]$ oc get job
NAME                  COMPLETIONS   DURATION   AGE
hello-lixiaohui-job   1/1           12s        5m15s
```

```bash
[student@workstation ~]$ oc create -f job.yml
[student@workstation ~]$ oc get pod
NAME                        READY   STATUS      RESTARTS   AGE
hello-lixiaohui-job-fqh4t   0/1     Completed   0          16s

[student@workstation ~]$ oc logs hello-lixiaohui-job-fqh4t
hello lixiaohui
```

## Cron Job

cron 作业资源包括描述任务和调度的作业模板，Kubernetes cron 作业的调度规范衍生自 Linux cron 作业中的规范，我们来看看Linux里的cron知识

```bash
[student@workstation ~]$ cat /etc/crontab
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root

# For details see man 4 crontabs

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name  command to be executed
```

以下是 cron 作业规范的几个示例：

| 调度规范            | 描述          |
| --------------- | ----------- |
| `​ 0 ​ 0 * * *` | 每天午夜运行指定的任务 |
| `​ 0 ​ 0 * * 7` | 每周日运行指定的任务  |
| `​ 0 ​ * * * *` | 每小时运行指定的任务  |
| `*/4 * * * *`   | 每四分钟运行指定的任务 |

我们来创建一个crontjob

这个cronjob每分钟会输出一句话

```yaml
cat > cronjob.yml <<EOF
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjobtest
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: registry.ocp4.example.com:8443/openshift/origin-cli:4.12
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello lixiaohui again
          restartPolicy: OnFailure
EOF
```

```bash
[student@workstation ~]$ oc create -f cronjob.yml
```

这个不会那么快的看到结果，因为要到1分钟后，它才会运行

```bash
[student@workstation ~]$ oc get cronjobs.batch
NAME          SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjobtest   */1 * * * *   False     0        <none>          18s
```

再看的时候，已经有了上一次的调度时间，并可以看到pod正常运行

```bash
[student@workstation ~]$ oc get cronjobs.batch
NAME          SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjobtest   */1 * * * *   False     0        12s             63s

[student@workstation ~]$ oc get pod
NAME                         READY   STATUS      RESTARTS   AGE
cronjobtest-28914331-r7rdz   0/1     Completed   0          16s

[student@workstation ~]$ oc logs cronjobtest-28914331-r7rdz
Sun Dec 22 09:31:01 UTC 2024
Hello lixiaohui again
```


