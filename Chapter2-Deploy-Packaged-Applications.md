```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 部署打包应用

# 课程目标

- 从存储在 OpenShift 模板中的资源清单部署应用及其依赖项。​

- 从打包为 Helm 图标的资源清单部署和更新应用。​

# OpenShift 模板

模板是一种 Kubernetes 自定义资源，描述了一组 Kubernetes 资源配置。​模板可以有参数。​可以通过处理模板并提供参数的值，从模板创建一组相关的 Kubernetes 资源。模板资源是红帽为 OpenShift 提供的 Kubernetes 扩展。​Cluster Samples Operator 填充 `openshift` 命名空间中的模板（和镜像流），当然你也可以自己创建自己的模板。

## 列出模板

Cluster Samples Operator 提供的模板位于 `openshift` 命名空间中

```bash
[student@workstation ~]$ oc get templates -n openshift
NAME                           DESCRIPTION                                                                        PARAMETERS        OBJECTS
cakephp-mysql-example          An example CakePHP application with a MySQL database. For more information ab...   21 (4 blank)      8
cakephp-mysql-persistent       An example CakePHP application with a MySQL database. For more information ab...   22 (4 blank)      9
dancer-mysql-example           An example Dancer application with a MySQL database. For more information abo...   18 (5 blank)      8
dancer-mysql-persistent        An example Dancer application with a MySQL database. For more information abo...   19 (5 blank)      9
django-psql-example            An example Django application with a PostgreSQL database. For more informatio...   19 (5 blank)      8
django-psql-persistent         An example Django application with a PostgreSQL database. For more informatio...   20 (5 blank)      9
httpd-example                  An example Apache HTTP Server (httpd) application that serves static content....   10 (3 blank)      5
mysql-ephemeral                MySQL database service, without persistent storage. For more information abou...   8 (3 generated)   3
mysql-persistent               MySQL database service, with persistent storage. For more information about u...   9 (3 generated)   4
nodejs-postgresql-example      An example Node.js application with a PostgreSQL database. For more informati...   18 (4 blank)      8
nodejs-postgresql-persistent   An example Node.js application with a PostgreSQL database. For more informati...   19 (4 blank)      9
postgresql-ephemeral           PostgreSQL database service, without persistent storage. For more information...   7 (2 generated)   3
postgresql-persistent          PostgreSQL database service, with persistent storage. For more information ab...   8 (2 generated)   4
```

## 查看模板完整的yaml清单以及参数

```yaml
[student@workstation ~]$ oc get templates -n openshift httpd-example -o yaml
apiVersion: template.openshift.io/v1
kind: Template
labels:
  app: httpd-example
  template: httpd-example
message: |-
  The following service(s) have been created in your project: ${NAME}.

  For more information about using this template, including OpenShift considerations, see https://github.com/sclorg/httpd-ex/blob/master/README.md.
metadata:
  annotations:
    description: An example Apache HTTP Server (httpd) application that serves static
      content. For more information about using this template, including OpenShift
      considerations, see https://github.com/sclorg/httpd-ex/blob/master/README.md.
    iconClass: icon-apache
    openshift.io/display-name: Apache HTTP Server
    openshift.io/documentation-url: https://github.com/sclorg/httpd-ex
    openshift.io/long-description: This template defines resources needed to develop
      a static application served by Apache HTTP Server (httpd), including a build
      configuration and application deployment configuration.
    openshift.io/provider-display-name: Red Hat, Inc.
    openshift.io/support-url: https://access.redhat.com
    samples.operator.openshift.io/version: 4.12.0
    tags: quickstart,httpd
    template.openshift.io/bindable: "false"
  creationTimestamp: "2023-11-13T17:44:58Z"
  labels:
    samples.operator.openshift.io/managed: "true"
  name: httpd-example
  namespace: openshift
  resourceVersion: "328084"
  uid: 640c1f62-6a3c-44c3-b5d4-3b08705c483e
objects:
- apiVersion: v1
  kind: Service
  metadata:
    annotations:
      description: Exposes and load balances the application pods
    name: ${NAME}
  spec:
    ports:
    - name: web
      port: 8080
      targetPort: 8080
    selector:
      name: ${NAME}
- apiVersion: route.openshift.io/v1
  kind: Route
  metadata:
    name: ${NAME}
  spec:
    host: ${APPLICATION_DOMAIN}
    to:
      kind: Service
      name: ${NAME}
```

## 查询模板属性

```bash
[student@workstation ~]$ oc describe template httpd-example -n openshift
Name:           httpd-example
Namespace:      openshift
Created:        13 months ago
Labels:         samples.operator.openshift.io/managed=true
Description:    An example Apache HTTP Server (httpd) application that serves static content. For more information about using this template, including OpenShift considerations, see https://github.com/sclorg/httpd-ex/blob/master/README.md.
Annotations:    iconClass=icon-apache
                openshift.io/display-name=Apache HTTP Server
                openshift.io/documentation-url=https://github.com/sclorg/httpd-ex
                openshift.io/long-description=This template defines resources needed to develop a static application served by Apache HTTP Server (httpd), including a build configuration and application deployment configuration.
                openshift.io/provider-display-name=Red Hat, Inc.
                openshift.io/support-url=https://access.redhat.com
                samples.operator.openshift.io/version=4.12.0
                tags=quickstart,httpd
                template.openshift.io/bindable=false

Parameters:
    Name:               NAME
    Display Name:       Name
    Description:        The name assigned to all of the frontend objects defined in this template.
    Required:           true
    Value:              httpd-example

    Name:               NAMESPACE
    Display Name:       Namespace
    Description:        The OpenShift Namespace where the ImageStream resides.
    Required:           true
    Value:              openshift

    Name:               HTTPD_VERSION
    Display Name:       HTTPD Version
    Description:        Version of HTTPD image to be used (2.4-el8 by default).
    Required:           true
    Value:              2.4-el8

    Name:               MEMORY_LIMIT
    Display Name:       Memory Limit
    Description:        Maximum amount of memory the container can use.
    Required:           true
    Value:              512Mi

    Name:               SOURCE_REPOSITORY_URL
    Display Name:       Git Repository URL
    Description:        The URL of the repository with your application source code.
    Required:           true
    Value:              https://github.com/sclorg/httpd-ex.git

    Name:               SOURCE_REPOSITORY_REF
    Display Name:       Git Reference
    Description:        Set this to a branch name, tag or other ref of your repository if you are not using the default branch.
    Required:           false
    Value:              <none>

    Name:               CONTEXT_DIR
    Display Name:       Context Directory
    Description:        Set this to the relative path to your project if it is not in the root of your repository.
    Required:           false
    Value:              <none>

    Name:               APPLICATION_DOMAIN
    Display Name:       Application Hostname
    Description:        The exposed hostname that will route to the httpd service, if left blank a value will be defaulted.
    Required:           false
    Value:              <none>

    Name:               GITHUB_WEBHOOK_SECRET
    Display Name:       GitHub Webhook Secret
    Description:        Github trigger secret.  A difficult to guess string encoded as part of the webhook URL.  Not encrypted.
    Required:           false
    Generated:          expression
    From:               [a-zA-Z0-9]{40}

    Name:               GENERIC_WEBHOOK_SECRET
    Display Name:       Generic Webhook Secret
    Description:        A secret string used to configure the Generic webhook.
    Required:           false
    Generated:          expression
    From:               [a-zA-Z0-9]{40}


Object Labels:  app=httpd-example,template=httpd-example

Message:        The following service(s) have been created in your project: ${NAME}.

                For more information about using this template, including OpenShift considerations, see https://github.com/sclorg/httpd-ex/blob/master/README.md.

Objects:
    Service                             ${NAME}
    Route.route.openshift.io            ${NAME}
    ImageStream.image.openshift.io      ${NAME}
    BuildConfig.build.openshift.io      ${NAME}
    DeploymentConfig.apps.openshift.io  ${NAME}
```

上面的内容太多了，可以用下面的方法，仅列出模板使用的参数即可

```bash
[student@workstation ~]$ oc process --parameters httpd-example -n openshift
NAME                     DESCRIPTION                                                                                               GENERATOR           VALUE
NAME                     The name assigned to all of the frontend objects defined in this template.                                                    httpd-example
NAMESPACE                The OpenShift Namespace where the ImageStream resides.                                                                        openshift
HTTPD_VERSION            Version of HTTPD image to be used (2.4-el8 by default).                                                                       2.4-el8
MEMORY_LIMIT             Maximum amount of memory the container can use.                                                                               512Mi
SOURCE_REPOSITORY_URL    The URL of the repository with your application source code.                                                                  https://github.com/sclorg/httpd-ex.git
SOURCE_REPOSITORY_REF    Set this to a branch name, tag or other ref of your repository if you are not using the default branch.
CONTEXT_DIR              Set this to the relative path to your project if it is not in the root of your repository.
APPLICATION_DOMAIN       The exposed hostname that will route to the httpd service, if left blank a value will be defaulted.
GITHUB_WEBHOOK_SECRET    Github trigger secret.  A difficult to guess string encoded as part of the webhook URL.  Not encrypted.   expression          [a-zA-Z0-9]{40}
GENERIC_WEBHOOK_SECRET   A secret string used to configure the Generic webhook.                                                    expression          [a-zA-Z0-9]{40}
```

## 使用模板

`oc new-app` 命令具有 `--template` 选项，可以直接从 `openshift` 项目部署模板资源。

```bash
[student@workstation ~]$ oc new-app --template=httpd-example -p NAME=lixiaohui
--> Deploying template "default/httpd-example" to project default

     Apache HTTP Server
     ---------
     An example Apache HTTP Server (httpd) application that serves static content. For more information about using this template, including OpenShift considerations, see https://github.com/sclorg/httpd-ex/blob/master/README.md.

     The following service(s) have been created in your project: lixiaohui.

     For more information about using this template, including OpenShift considerations, see https://github.com/sclorg/httpd-ex/blob/master/README.md.

     * With parameters:
        * Name=lixiaohui
        * Namespace=openshift
        * HTTPD Version=2.4-el8
        * Memory Limit=512Mi
        * Git Repository URL=https://github.com/sclorg/httpd-ex.git
        * Git Reference=
        * Context Directory=
        * Application Hostname=
        * GitHub Webhook Secret=D5iaWBYs7neNmNirw8YuEB7SIXgfPre6HstjXpK5 # generated
        * Generic Webhook Secret=TvoiO2eYrST2tLxnhjsK0F4kBmJKeuGXF44sxoCC # generated

--> Creating resources ...
    service "lixiaohui" created
    route.route.openshift.io "lixiaohui" created
    imagestream.image.openshift.io "lixiaohui" created
    buildconfig.build.openshift.io "lixiaohui" created
    deploymentconfig.apps.openshift.io "lixiaohui" created
--> Success
    Access your application via route 'lixiaohui-default.apps.ocp4.example.com'
    Build scheduled, use 'oc logs -f buildconfig/lixiaohui' to track its progress.
    Run 'oc status' to view your app.
```

查看部署的过程

```bash
[student@workstation ~]$ oc status --suggest
In project default on server https://api.ocp4.example.com:6443

svc/openshift - kubernetes.default.svc.cluster.local
svc/kubernetes - 172.30.0.1:443 -> 6443

http://lixiaohui-default.apps.ocp4.example.com (svc/lixiaohui)
  dc/lixiaohui deploys istag/lixiaohui:latest <-
    bc/lixiaohui source builds https://github.com/sclorg/httpd-ex.git on openshift/httpd:2.4-el8
    deployment #1 deployed about a minute ago - 1 pod

deployment/lixiaohui-deployment deploys nginx
  deployment #1 running for 3 hours - 0/3 pods

deployment/d deploys mysql
  deployment #3 running for 3 hours - 0/1 pods
  deployment #2 deployed 3 hours ago - 0/1 pods
  deployment #1 deployed 3 hours ago
```

不过需要注意，new-app不能用于更新应用

## 生成确切的资源清单文件

用模板和参数，可以生成最终的资源清单文件，用于版本控制

```yaml
[student@workstation ~]$ oc process httpd-example -n openshift -p NAME=lxh-name -o yaml > myyaml.yaml
[student@workstation ~]$ cat myyaml.yaml
apiVersion: v1
items:
- apiVersion: v1
  kind: Service
  metadata:
    annotations:
      description: Exposes and load balances the application pods
    labels:
      app: httpd-example
      template: httpd-example
    name: lxh-name
  spec:
    ports:
    - name: web
      port: 8080
      targetPort: 8080
    selector:
      name: lxh-name
- apiVersion: route.openshift.io/v1
  kind: Route
  metadata:
    labels:
      app: httpd-example
      template: httpd-example
    name: lxh-name
  spec:
```

使用-p选项将多个参数放在命令行的话，参数不方便保存，所以参数也可以放在文件中

生成参数文件

```bash
cat > myargs.file <<-EOF
NAME=lixiaohui
EOF
```

```bash
[student@workstation ~]$ oc process httpd-example -n openshift --param-file=myargs.file > mylist.yml
```

或者直接创建出线上资源

```bash
[student@workstation ~]$ oc process httpd-example -n openshift --param-file=myargs.file | oc apply -f -
```

## 删除模板

删除之前，我导出了一份用于备份

```bash
[student@workstation ~]$ oc get templates -n openshift httpd-example -o yaml > my-template.yml
[student@workstation ~]$
[student@workstation ~]$ oc delete template -n openshift httpd-example
template.template.openshift.io "httpd-example" deleted
```

## 创建新模板

用刚才的备份来生成新的模板，我创建到我自己的namespace中

```bash
[student@workstation ~]$ sed -i 's|namespace: openshift|namespace: lxh|g' my-template.yml

[student@workstation ~]$ oc create -f my-template.yml
template.template.openshift.io/httpd-example created
```

# Helm 图表(Charts)

Helm 是一个开源应用，可帮助管理 Kubernetes 应用的生命周期。​

Helm 引入了​*Charts*​的概念。Helm 图表定义了您可以部署的 Kubernetes 资源。 图表是具有定义结构的⽂件集合。 这些⽂件包括图表元数据（如图表名称或版本）、资源定义和⽀持材料。

最⼩ Helm 图表的结构要包括以下几项：

1. Chart.yaml：文件包含图表元数据，如图表的名称和版本

2. templates目录包含定义应用资源（如部署）的文件

3. values.yaml文件包含图表的默认值

Helm中有几个概念

1. Charts，是 `helm` 命令部署的打包应用

2. Releases，是部署图表的结果。​您可以将一个图表部署到同一集群多次。​每个部署都是一个不同的发行版。​

3. Versions，一个 Helm 图表可以有多个版本。​图表作者可以发布图表的更新，以适应后续的应用版本、​引入新功能或修复问题。

## 添加Charts仓库

helm必须要将仓库添加到本地，才能从仓库中安装软件

```bash
[student@workstation ~]$ helm repo add do280-repo   http://helm.ocp4.example.com/charts
[student@workstation ~]$ helm repo list
NAME            URL
do280-repo      http://helm.ocp4.example.com/charts
```

## 搜索仓库中的helm包

如果不加--versions，同一个软件有不同版本时，只显示最新版

```bash
[student@workstation ~]$ helm search repo --versions
```

## 查询Charts基本信息与values

```bash
[student@workstation ~]$ helm show chart do280-repo/etherpad
apiVersion: v2
appVersion: latest
description: A Helm chart for etherpad lite
home: https://github.com/redhat-cop/helm-charts
icon: https://pbs.twimg.com/profile_images/1336377123964145665/2gTadaDt_400x400.jpg
maintainers:
- name: eformat
name: etherpad
type: application
version: 0.0.7
```

```yaml
[student@workstation ~]$ helm show values do280-repo/etherpad
# Default values for etherpad.
replicaCount: 1

defaultTitle: "Labs Etherpad"
defaultText: "✍️ Assign yourself a user and share your ideas! ✍️"

image:
  repository: etherpad
  name:
  tag:
  pullPolicy: IfNotPresent

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

podSecurityContext: {}
securityContext: {}

service:
  type: ClusterIP
  port: 9001

ingress:
  enabled: false
  hosts:
    - name: etherpad.organization.com
  annotations: {}

route:
  enabled: true
  host: null
  targetPort: http
```

## 安装Helm Chart

安装helm包，需要创建values.yaml文件为软件提供参数值

```yaml
cat > values.yaml <<-EOF
image:
  repository: registry.ocp4.example.com:8443/etherpad
  name: etherpad
  tag: 1.8.18
route:
  host: development-etherpad.apps.ocp4.example.com
EOF
```

安装0.0.6版本

```bash
[student@workstation ~]$ helm install lixiaohui-app do280-repo/etherpad -f values.yaml --version 0.0.6
NAME: lixiaohui-app
LAST DEPLOYED: Thu Dec 19 01:24:44 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

看看部署的效果

```bash
[student@workstation ~]$ helm list
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
lixiaohui-app   default         1               2024-12-19 01:24:44.705048565 -0500 EST deployed        etherpad-0.0.6  latest

[student@workstation ~]$ oc get route
NAME                     HOST/PORT                                    PATH   SERVICES                 PORT    TERMINATION     WILDCARD
lixiaohui                lixiaohui-default.apps.ocp4.example.com             lixiaohui                <all>                   None
lixiaohui-app-etherpad   development-etherpad.apps.ocp4.example.com          lixiaohui-app-etherpad   http    edge/Redirect   None

[student@workstation ~]$ curl -s https://development-etherpad.apps.ocp4.example.com | grep -i Labs
        <title>Labs Etherpad</title>
```

## 升级Chart

helm upgrade 命令可以将更改应用到现有发行版，例如更新值或图表版本。这里我们升级到0.0.7

```bash
[student@workstation ~]$ helm list
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
lixiaohui-app   default         1               2024-12-19 01:24:44.705048565 -0500 EST deployed        etherpad-0.0.6  latest
[student@workstation ~]$ helm upgrade lixiaohui-app do280-repo/etherpad   -f values.yaml --version 0.0.7
Release "lixiaohui-app" has been upgraded. Happy Helming!
NAME: lixiaohui-app
LAST DEPLOYED: Thu Dec 19 01:33:01 2024
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
[student@workstation ~]$ helm list
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
lixiaohui-app   default         2               2024-12-19 01:33:01.298788916 -0500 EST deployed        etherpad-0.0.7  latest
```

服务照样在线

```bash
[student@workstation ~]$ curl -s https://development-etherpad.apps.ocp4.example.com | grep -i Labs
        <title>Labs Etherpad</title>
```

## 回滚Chart

天有不测风云，有时候需要回退，我们回到上一版

```bash
[student@workstation ~]$ helm history lixiaohui-app
REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION
1               Thu Dec 19 01:24:44 2024        superseded      etherpad-0.0.6  latest          Install complete
2               Thu Dec 19 01:33:01 2024        deployed        etherpad-0.0.7  latest          Upgrade complete
[student@workstation ~]$ helm rollback lixiaohui-app 1
Rollback was a success! Happy Helming!
```


