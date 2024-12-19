```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 红帽 OpenShift 管理二：配置生产性Kubernetes集群

DO280 侧重于配置 OpenShift 的多租户和安全功能。​DO280 还讲授如何基于operator来管理 OpenShift 附加组件。​本课程基于红 帽® OpenShift® 容器平台 4.12 搭建。​

## 课程目标

- 配置和管理 OpenShift 集群，以跨多个应用和开发团队保持安全性和可靠性。​

- 配置身份验证、​授权和资源配额。​

- 通过网络政策和 TLS 安全性（HTTPS）保护网络流量。​

- 使用 HTTP 和 TLS 以外的协议公开应用，并将应用附加到多宿主网络。​

- 管理 OpenShift 集群更新和 Kubernetes operator 更新。​

## 观众

- 对 OpenShift 集群、​应用、​用户和附加组件的持续管理感兴趣的系统管理员。​

- 对 Kubernetes 集群持续维护和故障排除感兴趣的站点可靠性工程师。​

- 有兴趣了解 OpenShift 集群安全性的系统和软件架构师。​

## 先决条件

- 红帽认证系统管理员（红帽企业 Linux 中的 RHCSA）认证或同等经验。 

- <mark>已经学完红帽 OpenShift 一：容器和 Kubernetes（DO180 v4.12）课程或同等经验</mark>。<mark>本课程原内容不会涉及到怎么安装OpenShift和Kubernetes</mark>

## 课堂环境介绍

![](https://gitee.com/cnlxh/do280/raw/master/images/chapter0/Single_node_cluster.svg)

<mark>workstation</mark> 虚拟机 (VM)是<mark>唯⼀装有图形桌⾯</mark>的虚拟机

本课堂中使用了红 帽 OpenShift 容器平台（RHOCP）  4.12 单节点（SNO）裸机 UPI 安装。​RHOCP 集群的基础架构系统位于 ocp4.example.com DNS 域中

**课堂计算机**

| 计算机名称                       | IP 地址          | 角色                                               |
| --------------------------- | -------------- | ------------------------------------------------ |
| bastion.lab.example.com     | 172.25.250.254 | 将虚拟机链接到中央服务器的路由器                                 |
| classroom.lab.example.com   | 172.25.252.254 | 托管所需课堂资料的服务器                                     |
| idm.ocp4.example.com        | 192.168.50.40  | 用于集群身份验证和授权支持的身份管理服务器                            |
| master01.ocp4.example.com   | 192.168.50.10  | RHOCP 单节点（SNO）集群                                 |
| registry.ocp4.example.com   | 192.168.50.50  | 注册表服务器，用于为集群提供私有注册表和 GitLab 服务                   |
| utility.lab.example.com     | 192.168.50.254 | 用于提供 RHOCP 集群所需支持服务的服务器，包括 DHCP、​NFS 以及通向集群网络的路由 |
| workstation.lab.example.com | 172.25.250.9   | 学员使用的图形工作站                                       |

`bastion` 的主要功能是充当连接学员计算机的网络和课堂网络之间的路由器。​如果 `bastion` 关闭，则其他学员计算机无法正常工作，甚至可能在启动过程中挂起。​

`utility` 系统充当连接 RHOCP 集群计算机的网络与学员网络之间的路由器。​如果 `utility` 关闭，则 RHOCP 集群无法正常工作，甚至可能在启动过程中挂起。​

在部分练习中，课堂包含一个孤立的网络。​只有 `utility` 系统和集群连接到此网络。​

课堂中有几个系统提供支持服务。​`classroom` 服务器托管动手实践活动中使用的软件和实验材料。​`registry` 服务器是私有的红 帽 Quay 容器注册表，用于托管用于动手实践活动的容器镜像。​有关如何使用这些服务器的信息将在这些活动的说明中提供。​

`master01` 系统充当 RHOCP 集群的控制平面和计算节点。​集群使用 `registry` 系统作为自己的私有容器镜像注册表和 GitLab 服务器。​`idm` 系统为 RHOCP 集群提供 LDAP 服务，以提供身份验证和授权支持。​

学员使用 `workstation` 计算机访问专用 RHOCP 集群，学员具有集群管理员特权。​

**RHOCP 访问方式**

| 访问方式         | 端点                                                      |
| ------------ | ------------------------------------------------------- |
| Web 控制台      | https://console-openshift-console.apps.ocp4.example.com |
| API          | https://api.ocp4.example.com:6443                       |
| registry 服务器 | https://registry.ocp4.example.com:8443                  |

RHOCP 集群有一个标准用户帐户 `developer`，其密码为 `developer`。​管理帐户 `admin` 的密码为 `redhatocp`。​

注册表配置有用户帐户 `developer` ，密码为 `developer`
