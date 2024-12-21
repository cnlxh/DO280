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

## 授予scc到服务账号

从公共容器注册表下载的容器镜像（如 Docker Hub）在使用 `restricted` SCC 时可能无法运行。​例如，需要以特定用户 ID 身份运行的容器镜像可能会失败，因为 `restricted` SCC 使用随机用户 ID 运行容器。​在端口 80 或端口 443 上侦听的容器镜像可能会因为相关原因而失败。​`restricted` SCC 使用的随机用户 ID 无法启动侦听特权网络端口（端口号小于 1024）的服务。

这种情况下，就需要创建服务账号，然后绑定`anyuid`，然后将这个服务账号绑定到deployment即可

```bash
[student@workstation ~]$ oc create serviceaccount lxh-sa
serviceaccount/lxh-sa created
```

```bash
[student@workstation ~]$ oc adm policy add-scc-to-user anyuid -z lxh-sa
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "lxh-sa"
```

```bash
[student@workstation ~]$ oc set serviceaccount deployment/nginx-deployment lxh-serviceaccount
deployment.apps/nginx-deployment serviceaccount updated
```


