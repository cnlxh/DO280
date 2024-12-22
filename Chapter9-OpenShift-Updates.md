```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# OpenShift 更新

# 课程目标

- 描述集群更新过程。​

- 识别使用已弃用 Kubernetes API 的应用。​

- 使用 Web 控制台和 CLI，更新 OLM 托管的operator。​

# 集群更新过程

## 集群更新简介

红帽 OpenShift 容器平台 4 通过使用红帽企业 Linux CoreOS 添加了许多新功能。​红帽发布了一个新的软件分发系统，提供了更新集群和底层操作系统的最佳升级路径。​  借助这种新的分发系统，<mark>OpenShift 集群可以执行空中更新</mark>（OTA）。​

这个能够 OTA 的软件分发系统可以管理控制器清单、​集群角色以及将集群更新为特定版本所需的任何其他资源。​借助这一功能，集群可以无缝运行最新版本。​OTA 使集群能够在新功能可用时及时使用，包括最新的漏洞修复和安全补丁。​OTA 显著减少了因升级而造成的停机时间。​

### candidate 通道

*candidate* 通道提供更新，以测试下一版本 OpenShift 容器平台中的功能接受度。​release candidate 版本需要进一步检查，在符合质量标准时会提升到 *fast* 或 *stable* 通道。<mark>​红帽不支持仅在 *candidate* 通道中列出的更新。</mark>

### fast 通道

一旦红帽宣布给定版本为 general availability 发布，*fast* 通道就会提供更新。​红帽支持此通道中发布的更新，<mark>它最适合开发和 QA 环境。</mark>​

### stable 通道

红帽支持团队和站点可靠性工程（SRE）团队会通过 *fast* 通道的更新来监控运作中的集群。​如果运作中的集群通过了额外的测试和验证，则在 *stable* 通道中会启用 *fast* 通道中的更新。​红帽支持此通道中发布的更新，<mark>它最适合生产环境。​</mark>

### 扩展更新支持通道

从 OpenShift 容器平台 4.8 开始，红帽会将所有偶数编号的次要版本（如 4.8、​4.10 和 4.12）标记为扩展更新支持（EUS）版本。​

<mark>为确保集群的稳定性和正确的支持级别，您不能在 *stable* 和 *fast* 通道之间来回切换。</mark>

## 暂停健康检测

在升级过程中，集群中的节点可能会暂时变得不可用。​对于 worker 节点，计算机健康检查可能会将此类节点识别为不健康，并重新启动它们。​为避免重新启动此类节点，请在更新集群之前暂停所有计算机健康检查资源。​

看看最多有多少机器可以不健康

- **NAME**: 资源的名称。在此例子中是 `machine-api-termination-handler`。

- **MAXUNHEALTHY**: 允许在任何时间点上的不健康机器的最大百分比。在这个例子中是 `100%`，这意味着理论上所有机器都可以是不健康的，而不会触发任何自动处理。

- **EXPECTEDMACHINES**: 预计存在的机器数。此字段在输出中未显示具体数量。

- **CURRENTHEALTHY**: 当前健康的机器数。输出中未显示具体数量。

```bash
[student@workstation ~]$ oc get machinehealthcheck -n openshift-machine-api
NAME                              MAXUNHEALTHY   EXPECTEDMACHINES   CURRENTHEALTHY
machine-api-termination-handler   100%
```

将 `cluster.x-k8s.io/paused` 注释添加到计算机健康检查资源，以在更新集群之前将其暂停。​

```bash
[student@workstation ~]$ oc annotate machinehealthcheck -n openshift-machine-api \
machine-api-termination-handler cluster.x-k8s.io/paused=""
```

在集群更新后，删除注释。​

```bash
[student@workstation ~]$ oc annotate machinehealthcheck -n openshift-machine-api \
machine-api-termination-handler cluster.x-k8s.io/paused-
```

## 更新界面展示

![](https://gitee.com/cnlxh/do280/raw/master/images/chapter9/cluster-settings.png)

# 检测已弃用 Kubernetes API 的使用情况

### Kubernetes API 弃用策略

Kubernetes API 版本根据功能成熟度进行分类（experimental、​pre-release 和 stable）。​

| API 版本   | 类别     | 描述              |
| -------- | ------ | --------------- |
| v1alpha1 | Alpha  | experimental 功能 |
| v1beta1  | beta   | pre-release 功能  |
| v1       | Stable | stable 功能，全面可用  |

举例来说，我们看看cronjob这个资源的API版本，它的版本就是v1，说明是稳定版

```bash
[student@workstation ~]$ oc api-resources | egrep '^NAME|cronjobs'
NAME                                  SHORTNAMES          APIVERSION                                    NAMESPACED   KIND
cronjobs                              cj                  batch/v1                                      true         CronJob
```

<mark>发布功能的 stable 版本时，beta 版本将标记为已弃用，并在三个 Kubernetes 版本后删除。</mark>

列出我们正在使用，但是将要弃用的API

```bash
[student@workstation ~]$ oc get apirequestcounts | awk '{if(NF==4){print $0}}'
NAME                                                                       REMOVEDINRELEASE   REQUESTSINCURRENTHOUR   REQUESTSINLAST24H
flowschemas.v1beta1.flowcontrol.apiserver.k8s.io                           1.26               0                       17
prioritylevelconfigurations.v1beta1.flowcontrol.apiserver.k8s.io           1.26               0                       9
```

## 集群更新前的显式确认

OpenShift 容器平台要求管理员必须先提供手动确认，然后才能将集群从版本 4.11 升级到 4.12。​此要求帮助防止在升级到 OpenShift 容器平台 4.12 后出现问题，其中工作负载、​工具或在集群上运行或与集群交互的其他组件仍在使用已删除的 API。​

管理员必须评估其集群中是否有任何工作负载正在使用已删除 API，并且迁移受影响的组件以使用适当的新 API 版本。​迁移后，管理员可以提供确认。​

执行下面的命令后，`admin-acks` ConfigMap 将被更新，表示管理员已确认了与 Kubernetes 1.25 版本有关的 API 移除通知，并准备升级到 OpenShift 4.12 版本。

```bash
oc patch configmap admin-acks -n openshift-config --type=merge \
  --patch '{"data":{"ack-4.11-kube-1.25-api-removals-in-4.12":"true"}}'
```

# 使用 OLM 更新operator

这个在界面上，如果operator有更新会提示，我们只需要在确认更新时，点击批准更新即可，对于自动审批的operator而言，在有更新时，会自动更新，无需管理员参与。


