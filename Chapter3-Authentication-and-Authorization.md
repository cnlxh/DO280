```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 身份验证和授权

# 课程目标

- 为 OpenShift 身份验证配置 HTPasswd 身份提供程序。​

- 定义基于角色的访问控制，并对用户应用权限。​

# 配置身份提供程序

OpenShift 用户和组相关的资源类型与定义

**用户**

在 OpenShift 容器平台架构中，用户是与 API 服务器交互的实体。​用户资源是对系统中参与者的表示。​通过将角色直接添加到用户或用户所属的组来分配权限。​

**身份**

身份资源记录来自特定用户和身份提供程序的成功身份验证尝试。​有关身份验证来源的任何数据都存储在身份中。​

**服务帐户**

在 OpenShift 中，当无法获取用户凭据时，应用可以独立与 API 进行通信。​为了保持普通用户凭据的完整性，凭据不会被共享，而是使用服务帐户。​服务帐户使您可以控制 API 访问，而无需借用普通用户的凭据。​

**组**

组代表一组特定的用户。​用户会被分配到组。​授权策略使用组来同时为多个用户分配许可权。​例如，要为 20 个用户分配对项目中的对象的访问权限，最好使用组，而不是单独为每个用户授予访问权限。​OpenShift 容器平台还提供由集群自动置备的系统组或虚拟组。​

**角色**

角色定义每个用户有权在指定资源类型上执行的 API 操作。​您可以通过将角色分配给用户、​组和服务帐户来授予权限。​

通常不会事先创建用户和身份资源。​OpenShift 通常会在使用 OAuth 成功进行交互式登录后自动创建这些资源。​

### 对 API 请求进行身份验证

身份验证和授权安全层使用户能够与集群进行交互。​当用户向 API 发出请求时，API 将用户与请求相关联。​身份验证层对用户进行身份验证。​身份验证成功后，授权层会接受或拒绝 API 请求。​授权层使用基于角色的访问控制（RBAC）策略来确定用户特权。​

OpenShift API 有两种方法来对请求进行身份验证：

- OAuth 访问令牌

- X.509 客户端证书

<mark>如果请求未提供访问令牌或证书，则身份验证层会为其分配 `system:anonymous` 虚拟用户，以及 `system:unauthenticated` 虚拟组。​</mark>

## 身份验证operator

OpenShift 容器平台提供身份验证operator，它将运行 OAuth 服务器。​<mark>OAuth 服务器在用户尝试对 API 进行身份验证时为他们提供 OAuth 访问令牌</mark>。​必须配置身份提供程序，并将其提供给 OAuth 服务器。​OAuth 服务器使用身份提供程序来验证请求者的身份。​服务器将用户与身份进行协调，然后为用户创建 OAuth 访问令牌。​成功登录后，OpenShift 自动创建身份和用户资源。​

**身份提供程序**

可以将 OpenShift OAuth 服务器配置为使用多个身份提供程序。​以下列表包含了常见的身份提供程序：

HTPasswd

针对机密验证用户名和密码，该机密中存储了使用 `htpasswd` 命令生成的凭据。​

Keystone

启用借助 OpenStack Keystone v3 服务器的共享身份验证。​

LDAP

配置 LDAP 身份提供程序，以使用简单的绑定身份验证对 LDAPv3 服务器验证用户名和密码。​

GitHub 或 GitHub Enterprise

配置 GitHub 身份提供程序，以根据 GitHub 或 GitHub Enterprises OAuth 身份验证服务器来验证用户名和密码。​

OpenID Connect

使用授权代码流，与 OpenID Connect 身份提供程序集成。​

## 集群管理员

​新安装的 OpenShift 集群提供了两种方法来通过集群管理员特权来验证 API 请求。​一种方法是<mark>使用 `kubeconfig` 文件，其中嵌入了永不过期的 X.509 客户端证书</mark>。​另一种方法是作为 `kubeadmin` 虚拟用户进行身份验证。​成功的身份验证会授予 OAuth 访问令牌。​

1. 在课堂环境中，`utility` 计算机将 `kubeconfig` 文件存储在 `/home/lab/ocp4/auth/kubeconfig` 中。​

2. 在课堂环境中，`utility` 计算机将 `kubeadmin` 用户的密码存储在 `/home/lab/ocp4/auth/kubeadmin-password` 文件中。​

```bash
[root@utility ~]# export KUBECONFIG=/home/lab/ocp4/auth/kubeconfig
[root@utility ~]# oc get nodes
NAME       STATUS   ROLES                         AGE    VERSION
master01   Ready    control-plane,master,worker   447d   v1.25.4+77bec7a
```

```bash
[root@utility ~]# oc login -u kubeadmin -p 8UgkW-u7pMu-223kK-PmNZH https://api.ocp4.example.com:6443
The server uses a certificate signed by an unknown authority.
You can bypass the certificate check, but any data you send to the server could be intercepted by others.
Use insecure connections? (y/n): y

WARNING: Using insecure TLS client config. Setting this option is not supported!

Login successful.

You have access to 72 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
Welcome! See 'oc help' to get started.

[root@utility ~]# oc get nodes
NAME       STATUS   ROLES                         AGE    VERSION
master01   Ready    control-plane,master,worker   447d   v1.25.4+77bec7a
```

<mark>这里要注意kubeadmin这个用户，这个是临时的超级管理员，在我们至少提供了一种身份认证提供程序后，这个用户最好删除，避免风险</mark>，不过在课程中，**切勿** 删除 `kubeadmin` 用户。​`kubeadmin` 用户对于课程实验架构至关重要。​删除 `kubeadmin` 用户会损坏实验环境

## 配置 HTPasswd 身份提供程序

在我们的课程中，我们用的是HTPasswd类型，HTPasswd 身份提供程序依照机密对用户进行验证，该机密中包含通过 Apache HTTP Server 项目中的 `htpasswd` 命令生成的用户名和密码。​只有集群管理员可以更改 HTPasswd 机密内的数据。​普通用户不能更改自己的密码。​

我们需要创建一些HTPasswd用户，然后给用户授予权限，并更新oauth服务器以便于使用HTPasswd类型

### 新建htpasswd用户

```bash
[student@workstation ~]$ htpasswd --help
 -c  Create a new file.
 -b  Use the password from the command line rather than prompting for it.
 -B  Force bcrypt encryption of the password (very secure).
```

这里创建了一个new-htpasswd.txt的文件，里面包含两个用户

```bash
[student@workstation ~]$ htpasswd -c -B -b new-htpasswd.txt lxh-admin lxhpass
Adding password for user lxh-admin
[student@workstation ~]$ htpasswd -B -b new-htpasswd.txt zhangsan zhangsanpass
Adding password for user zhangsan
[student@workstation ~]$ cat new-htpasswd.txt
lxh-admin:$2y$05$hGcuccbY8BGrmq5G58f3zOP2hz2w1/WqNPepJZ1oXsL9pUHoPOKzK
zhangsan:$2y$05$JFwYSQdeZmU1p1vv8vKweO9g2pApGcH8E7UHC7PwH5eZ6joLG4aua
```

### 分配集群超级管理员

分配权限之前，我们用创建机密的方式，先把这两个用户建出来

```bash
[student@workstation ~]$ oc login -u admin -p redhatocp https://api.ocp4.example.com:6443
[student@workstation ~]$ oc create secret generic my-htpass --from-file htpasswd=new-htpasswd.txt -n openshift-config
secret/my-htpass created
```

分配超级管理员

由于我们还没有将身份提供程序改为htpasswd，所以提示没有用户很正常，忽略即可

```bash
[student@workstation ~]$ oc adm policy add-cluster-role-to-user cluster-admin lxh-admin
Warning: User 'lxh-admin' not found
clusterrole.rbac.authorization.k8s.io/cluster-admin added: "lxh-admin"
```

### 添加HTPasswd认证方式

我们先导出服务器上正在生效的配置，稍微改改就行

```bash
[student@workstation ~]$ oc get oauth cluster -o yaml > my-oauth.yml
[student@workstation ~]$ cat my-oauth.yml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  annotations:
    include.release.openshift.io/ibm-cloud-managed: "true"
    include.release.openshift.io/self-managed-high-availability: "true"
    include.release.openshift.io/single-node-developer: "true"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"config.openshift.io/v1","kind":"OAuth","metadata":{"annotations":{},"name":"cluster"},"spec":{"identityProviders":[{"challenge":true,"htpasswd":{"fileData":{"name":"htpasswd-secret"}},"login":true,"mappingMethod":"claim","name":"htpasswd_provider","type":"HTPasswd"}]}}
    release.openshift.io/create-only: "true"
  creationTimestamp: "2023-09-28T14:08:46Z"
  generation: 3
  name: cluster
  ownerReferences:
  - apiVersion: config.openshift.io/v1
    kind: ClusterVersion
    name: version
    uid: 72d0e456-95f1-4410-926d-04980c8ba544
  resourceVersion: "332963"
  uid: c8ba3aad-8360-4c1b-9049-00410bc5d2eb
spec:
  identityProviders:
  - ldap:
      attributes:
        email:
        - mail
        id:
        - dn
        name:
        - cn
        preferredUsername:
        - uid
      bindDN: uid=admin,cn=users,cn=accounts,dc=ocp4,dc=example,dc=com
      bindPassword:
        name: ldap-secret
      ca:
        name: ca-config-map
      insecure: false
      url: ldap://idm.ocp4.example.com/cn=users,cn=accounts,dc=ocp4,dc=example,dc=com?uid
    mappingMethod: claim
    name: Red Hat Identity Management
    type: LDAP
```

在spec下的identityProviders中，添加新的类型

```bash
[student@workstation ~]$ cat my-oauth.yml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  annotations:
    include.release.openshift.io/ibm-cloud-managed: "true"
    include.release.openshift.io/self-managed-high-availability: "true"
    include.release.openshift.io/single-node-developer: "true"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"config.openshift.io/v1","kind":"OAuth","metadata":{"annotations":{},"name":"cluster"},"spec":{"identityProviders":[{"challenge":true,"htp                                                                                                                                                               asswd":{"fileData":{"name":"htpasswd-secret"}},"login":true,"mappingMethod":"claim","name":"htpasswd_provider","type":"HTPasswd"}]}}
    release.openshift.io/create-only: "true"
  creationTimestamp: "2023-09-28T14:08:46Z"
  generation: 3
  name: cluster
  ownerReferences:
  - apiVersion: config.openshift.io/v1
    kind: ClusterVersion
    name: version
    uid: 72d0e456-95f1-4410-926d-04980c8ba544
  resourceVersion: "332963"
  uid: c8ba3aad-8360-4c1b-9049-00410bc5d2eb
spec:
  identityProviders:
  - htpasswd:
      fileData:
        name: my-htpass
    mappingMethod: claim
    name: Lxh-Users
    type: HTPasswd
  - ldap:
      attributes:
        email:
        - mail
        id:
        - dn
        name:
        - cn
        preferredUsername:
        - uid
      bindDN: uid=admin,cn=users,cn=accounts,dc=ocp4,dc=example,dc=com
      bindPassword:
        name: ldap-secret
      ca:
        name: ca-config-map
      insecure: false
      url: ldap://idm.ocp4.example.com/cn=users,cn=accounts,dc=ocp4,dc=example,dc=com?uid
    mappingMethod: claim
    name: Red Hat Identity Management
    type: LDAP
```

这个会触发operator的更新，不会那么快，观察一下内容，确认更新过程已经完成

```bash
[student@workstation ~]$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.12.0    True        True          False      2s      OAuthServerDeploymentProgressing: deployment/oauth-openshift.openshift-authentication: observed generation is 13, desired generation is 14.
```

### 验证权限分配

经过验证，发现lxh-admin有权限获取节点，而zhangsan没有权限

```bash
[student@workstation ~]$ oc login -u lxh-admin -p lxhpass https://api.ocp4.example.com:6443
Login successful.

You have access to 72 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
[student@workstation ~]$ oc get nodes
NAME       STATUS   ROLES                         AGE    VERSION
master01   Ready    control-plane,master,worker   447d   v1.25.4+77bec7a

[student@workstation ~]$ oc get users
NAME        UID                                    FULL NAME       IDENTITIES
admin       bc98d46a-dd9f-4917-8246-089f10f95e75   Administrator   Red Hat Identity Management:dWlkPWFkbWluLGNuPXVzZXJzLGNuPWFjY291bnRzLGRjPW9jcDQsZGM9ZXhhbXBsZSxkYz1jb20
developer   12724778-65ba-411a-aa80-a9634228e116   . developer     Red Hat Identity Management:dWlkPWRldmVsb3Blcixjbj11c2Vycyxjbj1hY2NvdW50cyxkYz1vY3A0LGRjPWV4YW1wbGUsZGM9Y29t
lxh-admin   693dfbd1-5721-4ffe-b569-fa346675cf61                   Lxh-Users:lxh-admin
zhangsan    6563b503-367e-47fe-8a40-62dcb37e344a                   Lxh-Users:zhangsan
[student@workstation ~]$ oc get identities.user.openshift.io
NAME                                                                                                           IDP NAME                      IDP USER NAME                                                                      USER NAME   USER UID
Lxh-Users:lxh-admin                                                                                            Lxh-Users                     lxh-admin                                                                          lxh-admin   693dfbd1-5721-4ffe-b569-fa346675cf61

[student@workstation ~]$ oc login -u zhangsan -p zhangsanpass https://api.ocp4.example.com:6443
Login successful.

You don't have any projects. You can try to create a new project, by running

    oc new-project <projectname>

[student@workstation ~]$ oc get nodes
Error from server (Forbidden): nodes is forbidden: User "zhangsan" cannot list resource "nodes" in API group "" at the cluster scope

[student@workstation ~]$ oc get nodes
Error from server (Forbidden): nodes is forbidden: User "zhangsan" cannot list resource "nodes" in API group "" at the cluster scope
[student@workstation ~]$ oc get users
Error from server (Forbidden): users.user.openshift.io is forbidden: User "zhangsan" cannot list resource "users" in API group "user.openshift.io" at the cluster scope
[student@workstation ~]$ oc get identities.user.openshift.io
Error from server (Forbidden): identities.user.openshift.io is forbidden: User "zhangsan" cannot list resource "identities" in API group "user.openshift.io" at the clu
```

### 添加新用户到HTPasswd用户列表

添加用户，需要先把线上现有的用户导出来，然后新增，我们是模拟本地没有txt文件的情况下如何新增

这个--to可以不用加，默认是当前路径，--confirm如果不加，而当前路径存在有htpasswd文件时，会失败

```bash
[student@workstation ~]$ oc extract secret/my-htpass -n openshift-config --to=/home/student/ --confirm
/home/student/htpasswd
[student@workstation ~]$ cat /home/student/htpasswd
lxh-admin:$2y$05$hGcuccbY8BGrmq5G58f3zOP2hz2w1/WqNPepJZ1oXsL9pUHoPOKzK
zhangsan:$2y$05$JFwYSQdeZmU1p1vv8vKweO9g2pApGcH8E7UHC7PwH5eZ6joLG4aua
```

添加一个wangwu用户

```bash
[student@workstation ~]$ htpasswd -b -B htpasswd wangwu wangwupass
Adding password for user wangwu
[student@workstation ~]$ cat htpasswd
lxh-admin:$2y$05$hGcuccbY8BGrmq5G58f3zOP2hz2w1/WqNPepJZ1oXsL9pUHoPOKzK
zhangsan:$2y$05$JFwYSQdeZmU1p1vv8vKweO9g2pApGcH8E7UHC7PwH5eZ6joLG4aua
wangwu:$2y$05$AJorPPbJ1Z3cVci8IAV5zuQln4XBrwPrS5qQqfXpk3IGmd3w1CxOm
```

触发一次secret数据更新

供 HTPasswd 身份提供程序使用的机密需要在指定文件的路径之前添加 htpasswd= 前缀

--from-file后面的htpasswd是secret的key，后一个是文件

```bash
[student@workstation ~]$ oc set data secret/my-htpass --from-file htpasswd=htpasswd -n openshift-config
```

确认wangwu用户登录成功

```bash
[student@workstation ~]$ oc login -u wangwu -p wangwupass https://api.ocp4.example.com:6443
Login successful.
```

### 更新用户密码

更新wangwu用户的密码

```bash
[student@workstation ~]$ oc extract secret/my-htpass -n openshift-config --to=/home/student/ --confirm
/home/student/htpasswd
```

```bash
[student@workstation ~]$ cat htpasswd
lxh-admin:$2y$05$hGcuccbY8BGrmq5G58f3zOP2hz2w1/WqNPepJZ1oXsL9pUHoPOKzK
zhangsan:$2y$05$JFwYSQdeZmU1p1vv8vKweO9g2pApGcH8E7UHC7PwH5eZ6joLG4aua
wangwu:$2y$05$AJorPPbJ1Z3cVci8IAV5zuQln4XBrwPrS5qQqfXpk3IGmd3w1CxOm

[student@workstation ~]$ htpasswd -b -B htpasswd wangwu lixiaohuipass
Updating password for user wangwu

[student@workstation ~]$ cat htpasswd
lxh-admin:$2y$05$hGcuccbY8BGrmq5G58f3zOP2hz2w1/WqNPepJZ1oXsL9pUHoPOKzK
zhangsan:$2y$05$JFwYSQdeZmU1p1vv8vKweO9g2pApGcH8E7UHC7PwH5eZ6joLG4aua
wangwu:$2y$05$HoT.vyizHJotzDCQ1qjDLuJc7K3noHVyEej9UgFADcRrBc4VPOOf.
```

```bash
[student@workstation ~]$ oc set data secret/my-htpass --from-file htpasswd=htpasswd -n openshift-config
```

别忘了观察oc get co

### 删除用户

```bash
[student@workstation ~]$ oc extract secret/my-htpass -n openshift-config --to=/home/student/ --confirm
/home/student/htpasswd
```

删除用户可以手工vim删除这一条，或者用

```bash
[student@workstation ~]$ cat htpasswd
lxh-admin:$2y$05$hGcuccbY8BGrmq5G58f3zOP2hz2w1/WqNPepJZ1oXsL9pUHoPOKzK
zhangsan:$2y$05$JFwYSQdeZmU1p1vv8vKweO9g2pApGcH8E7UHC7PwH5eZ6joLG4aua
wangwu:$2y$05$HoT.vyizHJotzDCQ1qjDLuJc7K3noHVyEej9UgFADcRrBc4VPOOf.

[student@workstation ~]$ htpasswd -D htpasswd wangwu
Deleting password for user wangwu

[student@workstation ~]$ cat htpasswd
lxh-admin:$2y$05$hGcuccbY8BGrmq5G58f3zOP2hz2w1/WqNPepJZ1oXsL9pUHoPOKzK
zhangsan:$2y$05$JFwYSQdeZmU1p1vv8vKweO9g2pApGcH8E7UHC7PwH5eZ6joLG4aua
```

```bash
[student@workstation ~]$ oc set data secret/my-htpass --from-file htpasswd=htpasswd -n openshift-config
```

当出现需要删除用户的情形时，仅从身份提供程序中删除该用户是不够的。​还必须删除用户和身份资源

```bash
[student@workstation ~]$ oc delete identities.user.openshift.io Lxh-Users:
lxh-admin  wangwu     zhangsan
[student@workstation ~]$ oc delete identities.user.openshift.io Lxh-Users:wangwu
identity.user.openshift.io "Lxh-Users:wangwu" deleted
[student@workstation ~]$ oc delete user wangwu
user.user.openshift.io "wangwu" deleted
```

别忘了观察oc get co

### 新建组

```bash
[student@workstation ~]$ oc adm groups new lxh-group
group.user.openshift.io/lxh-group created
```

### 添加用户到组

```bash
[student@workstation ~]$ oc adm groups add-users lxh-group lxh-admin
group.user.openshift.io/lxh-group added: "lxh-admin"
```

查一下大家在哪个组

```bash
[student@workstation ~]$ oc get group
NAME                USERS
Default SMB Group
admins              Administrator
developer
editors
lxh-group           lxh-admin
ocpadmins           Administrator
ocpdevs             . developer
```

# 使用 RBAC 定义和应用权限

在红帽 OpenShift 中，RBAC 会确定用户是否可以在集群或项目中执行某些操作。​可以根据用户的职责级别，<mark>在两种角色类型之间选择：集群和本地。​</mark>

1. 集群的范围是跨多个project的

2. 本地是特指某个project

## 授权过程

授权过程由规则、​角色和绑定进行管理。​

| RBAC 对象 | 描述                   |
| ------- | -------------------- |
| 规则      | 允许对象或对象组执行的操作。​      |
| 角色      | 规则集。​用户和组可以与多个角色关联。​ |
| 绑定      | 将用户或组分配到某个角色。​       |

## RBAC 范围

根据用户的范围和责任，红帽 OpenShift 容器平台（RHOCP）定义了两组角色和绑定：<mark>集群角色和本地角色。​</mark>

| 角色级别 | 描述                              |
| ---- | ------------------------------- |
| 集群角色 | 具有此角色级别的用户或组可以管理 OpenShift 集群。​ |
| 本地角色 | 具有此角色级别的用户或组只能管理项目级别的元素。​       |

**<mark>集群角色绑定的优先级高于本地角色绑定。​</mark>**

## 默认角色

OpenShift 附带了一组默认的集群角色，可以在本地分配，或者分配到整个集群。​

| 默认角色               | 描述                                                                         |
| ------------------ | -------------------------------------------------------------------------- |
| `admin`            | 具有此角色的用户可以管理所有项目资源，包括向其他用户授予访问权限来访问项目。​                                    |
| `basic-user`       | 具有此角色的用户具有项目的读取访问权限。​                                                      |
| `cluster-admin`    | 具有此角色的用户拥有对集群资源的超级用户访问权限。​这些用户可以在集群上执行任何操作，并且拥有所有项目的完全控制权限。​               |
| `cluster-status`   | 具有此角色的用户可以获取集群状态信息。​                                                       |
| `edit`             | 具有此角色的用户可以创建、​更改和删除项目中的通用应用资源，如服务和部署。​这些用户无法操作管理资源，如限值范围和配额，也不能管理项目的访问权限。​ |
| `self-provisioner` | 具有此角色的用户可以创建项目。​这是集群角色，而非项目角色。​                                            |
| `view`             | 具有此角色的用户可以查看项目资源，但不能修改项目资源。​                                               |

### 用户类型

与 OpenShift 容器平台的交互与用户相关联。​OpenShift 容器平台用户对象代表用户，可以通过添加 `roles` 到该用户或通过角色绑定添加到该用户所属的组而授予其系统权限。​

**普通用户**

大多数交互式 OpenShift 容器平台用户都是普通用户，用 `User` 对象表示。​该类型用户代表有权访问平台的人员。​

**系统用户**

许多系统用户在定义基础架构时自动创建，主要用于使基础架构与 API 安全地交互。​系统用户包括有权访问一切资源的集群管理员、​各节点的用户、​路由器和注册表的用户，以及各种其他用户。​匿名系统用户默认用于未经身份验证的请求。​

系统用户名以 `system:` 前缀开头，如 `system:admin`、​`system:openshift-registry` 和 `system:node:node1.example.com`。​

**服务帐户**

服务帐户是与项目关联的系统用户。​工作负载可以使用服务帐户来调用 Kubernetes API。​

服务帐户通过 `ServiceAccount` 对象表示。​

系统用户名以 ``system:serviceaccount:*`namespace`*:`` 前缀开头，如 `system:serviceaccount:default:deployer` 和 `system:serviceaccount:accounting:builder`。​

## 授权案例

### 分配超级管理员

```bash
[student@workstation ~]$ oc adm policy add-cluster-role-to-user cluster-admin lxh-admin
clusterrole.rbac.authorization.k8s.io/cluster-admin added: "lxh-admin"
```

同理，只需要把to-user改成to-group即可授权给组

### 从用户身上回收集群权限

```bash
[student@workstation ~]$ oc adm policy remove-cluster-role-from-user cluster-admin zhangsan
```

同理，只需要把from-user改成from-group即可从组回收

### 分配本地角色

在default的project中，给lxh-admin分配edit角色

```bash
oc policy add-role-to-user edit lxh-admin -n default
```

### 确认谁有哪种权限

```bash
[student@workstation ~]$ oc adm policy who-can create deployment
resourceaccessreviewresponse.authorization.openshift.io/<unknown>

Namespace: default
Verb:      create
Resource:  deployments.apps

Users:  admin
        lxh-admin
        system:admin
        system:serviceaccount:metallb-system:manager-account
        system:serviceaccount:openshift-apiserver-operator:openshift-apiserver-operator
        system:serviceaccount:openshift-apiserver:openshift-apiserver-sa
        system:serviceaccount:openshift-authentication-operator:authentication-operator
        system:serviceaccount:openshift-authentication:oauth-openshift
        system:serviceaccount:openshift-cluster-storage-operator:cluster-storage-operator
        system:serviceaccount:openshift-cluster-version:default
        system:serviceaccount:openshift-config-operator:openshift-config-operator
        system:serviceaccount:openshift-controller-manager-operator:openshift-controller-manager-operator
        system:serviceaccount:openshift-etcd-operator:etcd-operator
        system:serviceaccount:openshift-etcd:installer-sa
        system:serviceaccount:openshift-infra:template-instance-controller
        system:serviceaccount:openshift-infra:template-instance-finalizer-controller
        system:serviceaccount:openshift-ingress-operator:ingress-operator
        system:serviceaccount:openshift-kube-apiserver-operator:kube-apiserver-operator
        system:serviceaccount:openshift-kube-apiserver:installer-sa
        system:serviceaccount:openshift-kube-apiserver:localhost-recovery-client
        system:serviceaccount:openshift-kube-controller-manager-operator:kube-controller-manager-operator
        system:serviceaccount:openshift-kube-controller-manager:installer-sa
        system:serviceaccount:openshift-kube-controller-manager:localhost-recovery-client
        system:serviceaccount:openshift-kube-scheduler-operator:openshift-kube-scheduler-operator
        system:serviceaccount:openshift-kube-scheduler:installer-sa
        system:serviceaccount:openshift-kube-scheduler:localhost-recovery-client
        system:serviceaccount:openshift-kube-storage-version-migrator-operator:kube-storage-version-migrator-operator
        system:serviceaccount:openshift-kube-storage-version-migrator:kube-storage-version-migrator-sa
        system:serviceaccount:openshift-machine-api:cluster-baremetal-operator
        system:serviceaccount:openshift-machine-config-operator:default
        system:serviceaccount:openshift-network-operator:default
        system:serviceaccount:openshift-oauth-apiserver:oauth-apiserver-sa
        system:serviceaccount:openshift-operator-lifecycle-manager:olm-operator-serviceaccount
        system:serviceaccount:openshift-service-ca-operator:service-ca-operator
        system:serviceaccount:openshift-storage:lvms-operator
Groups: ocpadmins
        system:cluster-admins
        system:masters
```


