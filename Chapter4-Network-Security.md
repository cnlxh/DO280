```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 网络安全

# 课程目标

- 允许并保护 OpenShift 集群内应用的网络连接。​

- 限制项目和 pod 之间的网络流量。​

- 配置和使用自动服务证书。​

# 利用 TLS 保护外部流量

OpenShift 容器平台提供了多种方式向外部网络公开您的应用。​您可以公开 HTTP 和 HTTPS 流量、​TCP 应用以及非 TCP 流量。​其中一些方法是服务​类型，如 `NodePort` 或负载平衡器，而另一些则使用自己的 API 资源，如 `Ingress` 和 `Route`。​

借助 OpenShift 路由，您可以向外部网络公开您的应用，从而通过可公开访问的唯一主机名称访问应用。​路由依赖于路由器插件，将来自公共 IP 的流量重定向到 pod。​

下图显示了路由如何公开在集群中作为 pod 运行的应用：

![](https://www.credclouds.com/images/network-sdn-routes-network.svg)

**保护route的方案一般有三种，注意用前两种**

**边缘**

使用边缘终止时，TLS 终止发生在路由器上，在流量路由到 pod 之前。​路由器服务于 TLS 证书，因此您必须将它们配置到路由中；否则，OpenShift 会将自己的证书分配到路由器，以用于 TLS 终止。​由于 TLS 是在路由器终止的，因此不会加密通过内部网络进行的、​从路由器到端点的连接。​

**直通**

借助直通终止，加密流量直接发送到目标 pod，而无需来自路由器的 TLS 终止。​在此模式中，应用负责为流量提供证书。​要支持应用和访问它的客户端之间的相互身份验证，直通是当前唯一一种方法。​

**再加密**

再加密是边缘终止的一种变体，即路由器通过证书终止 TLS，然后再加密它与端点的连接，这可能有不同的证书。​因此，完整的连接路径被加密，即使在内部网络上。​路由器使用健康检查来判断主机的真实性。​

## 边缘卸载

在边缘模式中使用路由时，客户端和路由器之间的流量会加密，但路由器和应用之间的流量则不会加密：

![](https://www.credclouds.com/images/network-sdn-routes-edge.svg)

这里需要用到HTTPS证书，我们在workstation上生成证书

### 证书生成

先生成root证书

```bash
[student@workstation ~]$ su -
```

```bash
openssl genrsa -out /etc/pki/tls/private/xiaohuiroot.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 -subj "/C=CN/ST=Shanghai/L=Shanghai/O=Company/OU=SH/CN=xiaohui.cn" \
-key /etc/pki/tls/private/xiaohuiroot.key \
-out /etc/pki/ca-trust/source/anchors/xiaohuiroot.crt
```

再生成证书请求，本次申请为*.apps.ocp4.example.com

```bash
openssl genrsa -out /etc/pki/tls/private/xiaohui.cn.key 4096
openssl req -sha512 -new \
-subj "/C=CN/ST=Shanghai/L=Shanghai/O=Company/OU=SH/CN=*.apps.ocp4.example.com" \
-key /etc/pki/tls/private/xiaohui.cn.key \
-out xiaohui.cn.csr
```

签发证书

```bash
openssl x509 -req -in xiaohui.cn.csr \
-CA /etc/pki/ca-trust/source/anchors/xiaohuiroot.crt \
-CAkey /etc/pki/tls/private/xiaohuiroot.key -CAcreateserial \
-out /etc/pki/tls/certs/xiaohui.cn.crt \
-days 3650
```

```bash
chmod +r /etc/pki/tls/certs/xiaohui.cn.crt
chmod +r /etc/pki/tls/private/xiaohui.cn.key
```

本地信任根证书

```bash
update-ca-trust
```

### 创建明文服务

先创建一个不加密的后端服务，此服务名为no-tls并工作在80端口

```yaml
cat > no-tls.yml <<-EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-http
  labels:
    app: todo-http
    name: todo-http
spec:
  replicas: 1
  selector:
    matchLabels:
      app: todo-http
      name: todo-http
  template:
    metadata:
      labels:
        app: todo-http
        name: todo-http
    spec:
      containers:
      - resources:
          limits:
            cpu: '0.5'
        image: registry.ocp4.example.com:8443/redhattraining/todo-angular:v1.1
        name: todo-http
        ports:
        - containerPort: 8080
          name: todo-http
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: todo-http
    name: todo-http
  name: no-tls
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    name: todo-http
EOF
```

```bash
[root@workstation ~]# oc create -f no-tls.yml
```

将服务暴露出来

```bash
oc expose svc no-tls --hostname no-tls.apps.ocp4.example.com
```

确认可以用不加密的方式访问

```bash
[student@workstation ~]$ oc get route todo-http
NAME        HOST/PORT                         PATH   SERVICES    PORT   TERMINATION   WILDCARD
todo-http   todo-http.apps.ocp4.example.com          todo-http   8080                 None
[student@workstation ~]$ curl -s no-tls.apps.ocp4.example.com | grep -i todo
<html lang="en" ng-app="todoItemsApp" ng-controller="appCtl">
    <title>ToDo app</title>
    <script type="text/javascript" src="assets/js/app/domain/todoitems.js"></script>
        <a class="navbar-brand" href="/">ToDo App</a>
```

### 创建加密的TLS服务

这次创建了一个主机为tls-only.apps.ocp4.example.com的服务地址

```bash
oc create route edge --service no-tls --hostname tls-only.apps.ocp4.example.com --key /etc/pki/tls/private/xiaohui.cn.key --cert /etc/pki/tls/certs/xiaohui.cn.crt
```

访问一下看看

```bash
curl -s https://tls-only.apps.ocp4.example.com | grep -i todo
<html lang="en" ng-app="todoItemsApp" ng-controller="appCtl">
    <title>ToDo app</title>
    <script type="text/javascript" src="assets/js/app/domain/todoitems.js"></script>
        <a class="navbar-brand" href="/">ToDo App</a>
```

## 使用直通路由保护应用安全

直通路由提供了一种安全的替代方式，因为应用会公开其 TLS 证书。​因此，流量在客户端和应用之间加密。

提供证书的最佳方式是使用 OpenShift TLS 机密。​通过挂载点将机密公开到容器中。​

下图显示了如何在容器中挂载 secret 资源。​然后，应用可以访问您的证书。​

![](https://www.credclouds.com/images/network-sdn-routes-passthrough.svg)

### 创建tls机密

再生成证书请求，本次申请为tls-pass.apps.ocp4.example.com

我这里复用代码，所以会覆盖前面的证书和私钥，如果需要请备份前面的

```bash
su -
```

```bash
openssl genrsa -out /etc/pki/tls/private/xiaohui.cn.key 4096
openssl req -sha512 -new \
-subj "/C=CN/ST=Shanghai/L=Shanghai/O=Company/OU=SH/CN=tls-pass.apps.ocp4.example.com" \
-key /etc/pki/tls/private/xiaohui.cn.key \
-out xiaohui.cn.csr
```

签发证书

```bash
openssl x509 -req -in xiaohui.cn.csr \
-CA /etc/pki/ca-trust/source/anchors/xiaohuiroot.crt \
-CAkey /etc/pki/tls/private/xiaohuiroot.key -CAcreateserial \
-out /etc/pki/tls/certs/xiaohui.cn.crt \
-days 3650
```

```bash
chmod +r /etc/pki/tls/certs/xiaohui.cn.crt
chmod +r /etc/pki/tls/private/xiaohui.cn.key
```

```bash
oc create secret tls tls-only --key /etc/pki/tls/private/xiaohui.cn.key --cert /etc/pki/tls/certs/xiaohui.cn.crt
```

### 创建加密的后端服务

创建一个名为todo-https-pass且工作在8443和80的端口上的服务

在这个服务中，我们引用了上面的tls机密

```bash
cat > tls-only-pass.yml <<-EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-https
  labels:
    app: todo-https
    name: todo-https
spec:
  replicas: 1
  selector:
    matchLabels:
      app: todo-https
      name: todo-https
  template:
    metadata:
      labels:
        app: todo-https
        name: todo-https
    spec:
      containers:
      - resources:
          limits:
            cpu: '0.5'
        image: registry.ocp4.example.com:8443/redhattraining/todo-angular:v1.2
        name: todo-https
        ports:
        - containerPort: 8080
          name: todo-http
        - containerPort: 8443
          name: todo-https
        volumeMounts:
        - name: tls-only
          readOnly: true
          mountPath: /usr/local/etc/ssl/certs
      resources:
        limits:
          memory: 64Mi
      volumes:
      - name: tls-only
        secret:
          secretName: tls-only
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: todo-https
    name: todo-https
  name: todo-https-pass
spec:
  ports:
  - name: https
    port: 8443
    protocol: TCP
    targetPort: 8443
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    name: todo-https
EOF
```

创建出服务

```bash
oc create -f tls-only-pass.yml
```

看看pod是否正常运行

```bash
[student@workstation ~]$ oc get -f tls-only-pass.yml
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/todo-https   0/1     1            0           7s

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
service/todo-https-pass   ClusterIP   172.30.64.247   <none>        8443/TCP,80/TCP   7s
[student@workstation ~]$ oc get pod
NAME                          READY   STATUS        RESTARTS   AGE
todo-https-69b956b947-5zp89   1/1     Running       0          32s
```

看看证书是否如期挂载到pod中

```bash
[student@workstation ~]$ oc describe pod todo-https-69b956b947-5zp89 | grep -A2 Mounts
    Mounts:
      /usr/local/etc/ssl/certs from tls-only (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-7nxs7 (ro)
```

### 创建直通安全路由

```bash
[student@workstation ~]$ oc create route passthrough tls-pass --service todo-https-pass --port 8443 --hostname tls-pass.apps.ocp4.example.com
route.route.openshift.io/tls-pass created
[student@workstation ~]$ oc get route
NAME       HOST/PORT                        PATH   SERVICES          PORT   TERMINATION   WILDCARD
tls-pass   tls-pass.apps.ocp4.example.com          todo-https-pass   8443   passthrough   None
```

访问https看看是否成功

```bash
[student@workstation ~]$ curl -vv -I  https://tls-pass.apps.ocp4.example.com
*   Trying 192.168.50.254:443...
* Connected to tls-pass.apps.ocp4.example.com (192.168.50.254) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
*  CAfile: /etc/pki/tls/certs/ca-bundle.crt
* TLSv1.0 (OUT), TLS header, Certificate Status (22):
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS header, Certificate Status (22):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS header, Certificate Status (22):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS header, Certificate Status (22):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS header, Certificate Status (22):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS header, Certificate Status (22):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS header, Finished (20):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS header, Certificate Status (22):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS header, Finished (20):
* TLSv1.2 (IN), TLS header, Certificate Status (22):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: C=CN; ST=Shanghai; L=Shanghai; O=Company; OU=SH; CN=tls-pass.apps.ocp4.example.com
*  start date: Dec 19 14:02:06 2024 GMT
*  expire date: Dec 17 14:02:06 2034 GMT
*  common name: tls-pass.apps.ocp4.example.com (matched)
*  issuer: C=CN; ST=Shanghai; L=Shanghai; O=Company; OU=SH; CN=*.apps.ocp4.example.com
*  SSL certificate verify ok.
```

# 配置网络策略

有时候，我们需要限制pod之间的访问，那此时，就需要配置网络策略，<mark>Kubernetes 网络政策使用标签而非 IP 地址来控制 pod 之间的网络流量。</mark> ，我们可以借助标签来实现管控入口和出口的流量

## 网络策略案例

我们来通过实验来验证网络策略

1. 在名为zhangsan的project中，创建两个pod，互相访问，测试是否成功

2. 在名为lixiaohui的project中，创建两个pod，互相访问，测试是否成功

3. 这两个project的pod互相访问，测试是否成功

4. 新建网络策略，验证同project和不同project的pod互访是否成功

### 新建zhangsan project的资源

```bash
oc new-project zhangsan
```

在zhangsan的namespace中，新建一个名为hello开头的pod

```bash
oc new-app --name hello --image registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:v1.0
```

在zhangsan的namespace中，新建一个名为test开头的pod

```bash
oc new-app --name test --image registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:v1.0
```

查看一下pod的ip

```bash
[root@workstation ~]# oc -n zhangsan get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP          NODE       NOMINATED NODE   READINESS GATES
hello-7c5959664f-4mwg7   1/1     Running   0          8m22s   10.8.0.81   master01   <none>           <none>
test-54d78b7-gwc8g       1/1     Running   0          6s      10.8.0.84   master01   <none>           <none>
```

### 测试zhangsan project中的互访

从test的pod中发起对hello的pod访问，发现可以成功，证明同project互访ok

```bash
[root@workstation ~]# oc rsh test-54d78b7-gwc8g curl 10.8.0.81:8080
<html>
  <body>
    <h1>Hello, world from nginx!</h1>
  </body>
</html>
```

### 新建lixiaohui project的资源

```bash
oc new-project lixiaohui
```

```bash
oc new-app --name sample-app --image registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:v1.0
```

检查是否能运行

```bash
[root@workstation ~]# oc get pod -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP          NODE       NOMINATED NODE   READINESS GATES
sample-app-564fdf8b8c-psjf8   1/1     Running   0          57s   10.8.0.85   master01   <none>           <none>
```

### 验证跨project的访问

在新建网络策略之前，我们先试试他们跨project目前是否能访问

结果显示，跨project访问没问题

```bash
[root@workstation ~]# oc rsh sample-app-564fdf8b8c-psjf8 curl 10.8.0.81:8080
<html>
  <body>
    <h1>Hello, world from nginx!</h1>
  </body>
</html>
```

### 开启白名单

经过测试，上面不管是否跨project，都能互相访问，我们来试试，让hello这个pod除lixiaohui这个project外，不让所有人访问，开启白名单方式

先看看hello这个pod有什么标签

```bash
[root@workstation ~]# oc describe pod -n zhangsan hello-7c5959664f-4mwg7 | grep -A 2 Labels
Labels:           deployment=hello
                  pod-template-hash=7c5959664f
Annotations:      k8s.ovn.org/pod-networks:
```

在zhangsan的namespace下， 创建除网络策略

```yaml
cat > only-allow-lixiaohui-project.yml <<-EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: only-allow-lixiaohui-project
  namespace: zhangsan
spec:
  podSelector:
    matchLabels:
      deployment: hello
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: lixiaohui
    ports:
    - protocol: TCP
      port: 8080
EOF
```

```bash
[root@workstation ~]# oc create -f only-allow-lixiaohui-project.yml
[root@workstation ~]# oc get networkpolicies.networking.k8s.io -n zhangsan
NAME                           POD-SELECTOR       AGE
only-allow-lixiaohui-project   deployment=hello   18s
```

测试一下是否只允许lixiaohui的project访问

好的，看上去访问成功

```bash
[root@workstation ~]# oc get pod -n lixiaohui -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP          NODE       NOMINATED NODE   READINESS GATES
sample-app-564fdf8b8c-psjf8   1/1     Running   0          15m   10.8.0.85   master01   <none>           <none>
[root@workstation ~]#
[root@workstation ~]#
[root@workstation ~]# oc get pod -n zhangsan -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP          NODE       NOMINATED NODE   READINESS GATES
hello-7c5959664f-4mwg7   1/1     Running   0          31m   10.8.0.81   master01   <none>           <none>
test-54d78b7-gwc8g       1/1     Running   0          23m   10.8.0.84   master01   <none>           <none>
[root@workstation ~]#
[root@workstation ~]#
[root@workstation ~]#
[root@workstation ~]# oc rsh sample-app-564fdf8b8c-psjf8 curl 10.8.0.81:8080
<html>
  <body>
    <h1>Hello, world from nginx!</h1>
  </body>
</html>
```

不过在新建网络策略之前，它就是成功的，所以我们试试从同一个project中的test这个pod中访问试试

发现卡住不动了，无法访问

```bash
[root@workstation ~]# oc -n zhangsan rsh test-54d78b7-gwc8g curl 10.8.0.81:8080
```

但是跨project访问lixiaohui是可以的

```bash
[root@workstation ~]# oc -n zhangsan rsh test-54d78b7-gwc8g curl 10.8.0.85:8080
<html>
  <body>
    <h1>Hello, world from nginx!</h1>
  </body>
</html>
```

# 利用 TLS 保护内部流量

## 零信任环境

**零信任环境**假定每次交互都以不受信任的状态开始。​<mark>用户只能访问明确允许的文件或对象；通信必须加密；并且客户端应用必须验证服务器的真实性</mark>。​零信任环境要求受信任的证书颁发机构（CA）签名用于加密流量的证书。​<mark>通过引用 CA 证书，应用可以使用已签名的证书以加密方式验证另一个应用的真实性</mark>。​

## service-ca控制器

OpenShift 提供 `service-ca` 控制器，用于为内部流量生成并签名服务证书。​`service-ca` 控制器创建一个机密，它填充已签名的证书和密钥。​deployment可以将此机密挂载为卷以使用已签名的证书。​此外，客户端应用需要信任 `service-ca` 控制器 CA。​

## 零信任案例

若要生成证书和密钥对，请将 `service.beta.openshift.io/serving-cert-secret-name=your-secret` 注释应用于服务。`service-ca` 控制器在同一命名空间中创建 `your-secret` 机密（如果不存在），并对其填充服务的已签名的证书和密钥对。​

### 创建后端服务

比如说，我们来为服务生成一个包含证书对的机密：lxh-secret

第一步，先把服务创建出来

```bash
oc project default
oc new-app --name hello --image registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:latest
oc get service

NAME              TYPE           CLUSTER-IP       EXTERNAL-IP                            PORT(S)           AGE
hello             ClusterIP      172.30.141.226   <none>                                 8080/TCP          13s
```

这个8080端口不方便访问，我们直接给它删了，改成443，下面的8888端口，是我们后期打算让pod工作在这个端口

```bash
oc delete service hello
oc expose deployment/hello --port 443 --target-port 8888
```

```bash
[student@workstation ~]$ oc get service hello
NAME    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
hello   ClusterIP   172.30.83.38   <none>        443/TCP   105s
```

### 为服务创建tls机密

第二步，为服务启用证书填充

```bash
[root@workstation ~]# oc annotate service hello service.beta.openshift.io/serving-cert-secret-name=lxh-secret
service/hello annotate
```

看看服务是否添加了这个注解，看上去多了3个cert相关的注解

```bash
[root@workstation ~]# oc describe service hello
Name:              hello
Namespace:         default
Labels:            app=hello
                   app.kubernetes.io/component=hello
                   app.kubernetes.io/instance=hello
Annotations:       openshift.io/generated-by: OpenShiftNewApp
                   service.alpha.openshift.io/serving-cert-signed-by: openshift-service-serving-signer@1706011744
                   service.beta.openshift.io/serving-cert-secret-name: lxh-secret
                   service.beta.openshift.io/serving-cert-signed-by: openshift-service-serving-signer@1706011744
```

我们来研究一下，这个自动生成的机密到底是什么

```bash
[root@workstation ~]# oc describe secrets lxh-secret
Name:         lxh-secret
Namespace:    default
Labels:       <none>
Annotations:  service.alpha.openshift.io/expiry: 2026-12-20T06:39:38Z
              service.beta.openshift.io/expiry: 2026-12-20T06:39:38Z
              service.beta.openshift.io/originating-service-name: hello
              service.beta.openshift.io/originating-service-uid: fc4aafa0-fa9f-4b7b-9d44-e52507b893a4

Type:  kubernetes.io/tls

Data
====
tls.crt:  2575 bytes
tls.key:  1675 bytes
```

证书是啥内容看看去

```bash
[root@workstation ~]# oc get secrets lxh-secret -o yaml
apiVersion: v1
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUR3VENDQXFtZ0F3SUJBZ0lJQVdmMURucWtFSmN3RFFZSktvWklodmNOQVFFTEJRQXdOakUwTURJR0ExVUUKQXd3cmIzQmxibk5vYVdaMExYTmtcmdkb25aMn
  tls.key: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUVBb011NGhDU2JvYmE3ZVIreC80MXUxd1Izd0I0aTdnTW82cGU4MTZnR2pGcklFbXJnCmRvbloyenZ1VlVLaG54Rm
kind: Secret
metadata:
  annotations:
    service.alpha.openshift.io/expiry: "2026-12-20T06:39:38Z"
    service.beta.openshift.io/expiry: "2026-12-20T06:39:38Z"
    service.beta.openshift.io/originating-service-name: hello
    service.beta.openshift.io/originating-service-uid: fc4aafa0-fa9f-4b7b-9d44-e52507b893a4
  creationTimestamp: "2024-12-20T06:39:38Z"
  name: lxh-secret
  namespace: default
  ownerReferences:
  - apiVersion: v1
    kind: Service
    name: hello
    uid: fc4aafa0-fa9f-4b7b-9d44-e52507b893a4
  resourceVersion: "170877"
  uid: 51726655-b1cb-407b-8c76-0f2551635b1e
type: kubernetes.io/tls
```

可以看到这个自动生成的机密中，的确包括了两个证书，我们只需要把证书挂载到我们的deployment中就可以了，我们先更新一下配置文件来包含tls部分

### 为后端服务准备tls配置文件

我们要读取的证书位于：/etc/pki/nginx/，让pod工作在8888端口

```bash
cat > nginx.conf <<-'EOF'
server {
    listen       8888 ssl http2 default_server;
    listen       [::]:8888 ssl http2 default_server;
    server_name  _;
    root         /usr/share/nginx/html;

    ssl_certificate "/etc/pki/nginx/server.crt";
    ssl_certificate_key "/etc/pki/nginx/private/server.key";
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout  10m;
    ssl_ciphers PROFILE=SYSTEM;
    ssl_prefer_server_ciphers on;

    include /etc/nginx/default.d/*.conf;

    location / {
    }

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}
EOF
```

把这个配置文件存为configmap，一会儿让deployment来挂载

```bash
[student@workstation ~]$ oc create configmap lxh-nginx-tls --from-file nginx.conf
configmap/lxh-nginx-tls created
```

用补丁文件的方法修改，比手工改更精准

```bash
cat > patch.yml <<-'EOF'
spec:
  template:
    spec:
      containers:
        - name: hello
          ports:
            - containerPort: 8888
          volumeMounts:
            - name: tls-config
              mountPath: /etc/nginx/conf.d/
            - name: server-secret
              mountPath: /etc/pki/nginx/
      volumes:
        - name: tls-config
          configMap:
            defaultMode: 420
            name: lxh-nginx-tls
        - name: server-secret
          secret:
            defaultMode: 420
            secretName: lxh-secret
            items:
              - key: tls.crt
                path: server.crt
              - key: tls.key
                path: private/server.key
EOF
```

来，打一下补丁，发现我们的pod age很新，是刚创建的

```bash
[student@workstation ~]$ oc patch deployment hello --patch-file patch.yml
[student@workstation ~]$ oc get pod
NAME                          READY   STATUS             RESTARTS       AGE
hello-74fbfc9fd-ppzpq         1/1     Running            0              28s
```

### 测试陌生pod访问

我们随便找个调试pod，看看能不能访问service

```bash
[student@workstation ~]$ oc get service hello
NAME    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
hello   ClusterIP   172.30.83.38   <none>        443/TCP   105s
```

发现，随便找个pod是无法建立安全连接的，但是用-k跳过证书验证可以访问

以下hello.default.svc的格式是k8s自带的

1. hello是服务名称

2. default是命名空间

3. svc表示这是一个服务

```bash
[student@workstation ~]$ oc debug -t

sh-4.4# curl https://hello.default.svc

curl: (60) SSL certificate problem: self signed certificate in certificate chain
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.

sh-4.4# curl https://hello.default.svc -k
<html>
  <body>
    <h1>Hello, world from nginx!</h1>
  </body>
</html>
```

### 为客户端准备CA证书

生成包含服务 CA 捆绑包的configmap，并使用它来创建 `client`

```bash
[student@workstation ~]$ oc create configmap ca-file
configmap/ca-file created
```

给这个configmap填充ca证书

```bash
[student@workstation ~]$ oc annotate configmap ca-file service.beta.openshift.io/inject-cabundle=true
configmap/ca-file annotate
[student@workstation ~]$ oc describe configmaps ca-file
Name:         ca-file
Namespace:    default
Labels:       <none>
Annotations:  service.beta.openshift.io/inject-cabundle: true

Data
====
service-ca.crt:
----
-----BEGIN CERTIFICATE-----
MIIDUTCCAjmgAwIBAgIIFAxCe1I9d3swDQYJKoZIhvcNAQELBQAwNjE0MDIGA1UE
Awwrb3BlbnNoaWZ0LXNlcnZpY2Utc2VydmluZy1zaWduZXJAMTcwNjAxMTc0NDAe
Fw0yNDAxMjMxMjA5MDNaFw0yNjAzMjMxMjA5MDRaMDYxNDAyBgNVBAMMK29wZW5z
aGlmdC1zZXJ2aWNlLXNlcnZpbmctc2lnbmVyQDE3MDYwMTE3NDQwggEiMA0GCSqG
SIb3DQEBAQUAA4IBDwAwggEKAoIBAQDBmQBMDkBshSKSBnnPSDOJFx5lN05uj1ij
QkdMkvZ77UF6+grNK7J2XjMUL7XGV1AB2dylr+/Ze8bv+zgaKVPuz33v5Qkq1Xq3
sGTCcvEOKFFQpNi2xvvz+SRxE3ZepSn466d8Yl8KAMOwUs41SV1hRWhMjDnQpJFY
1o6zBSF3NUHrjwpgdaoqxvpAZq0F12ZdjmP6kY64CvYujUxjpZ+WTwkqQHi/RXfL
F3JkbhX/dmMsMG4lRegMwzcUvrNHV89pqg82urLAXKpEdeaqiDq1rz5ImomTJUyY
nDFDQWApBS+ds++M364pKGktIIJT4S9bp1+HJWBMlVR3yRcglgD1AgMBAAGjYzBh
MA4GA1UdDwEB/wQEAwICpDAPBgNVHRMBAf8EBTADAQH/MB0GA1UdDgQWBBQbVoak
nlEWPiZHAj6Dv7RltXKfyDAfBgNVHSMEGDAWgBQbVoaknlEWPiZHAj6Dv7RltXKf
yDANBgkqhkiG9w0BAQsFAAOCAQEAS3XqN/+utRKusxadlawdR61lDh4CB8mMrhc6
AVTXqdAaE2j+T4xAXMQWkCAs82UDKuCUK2O+PTf9HrePiKLp3YWi+VoqFUoPiI++
KFl24z+kcOuTatJJdutZ3UN8bk4T24GINBlQNyN2P3HDGQ1FL2NLNc59xC5qxFGC
I1j4RwsxmDz2SIlPeselEMS/unqpWTAW4M9ZkMmvg7BeHsUnNtHJPtQwqYkaWlXn
df5REDU7IkMKuNdlxD5PehUjujGB29PaBubfxBxFlhFyKLWJ72nECEZdJAwku1sf
L+TYgUQf39c9HNo9tBmDg/XdOXGM0BUjDQ/rTo0mcbSmp6b/dg==
-----END CERTIFICATE-----


BinaryData
====

Events:  <none>
```

### 验证零信任效果

我们来启动一个带有此ca的客户端访问一下看看

```yaml
cat > pod-with-ca.yml <<-'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: client
spec:
  containers:
    - name: client
      image: registry.ocp4.example.com:8443/redhattraining/hello-world-nginx
      volumeMounts:
        - mountPath: /etc/pki/ca-trust/extracted/pem
          name: trusted-ca
  volumes:
    - configMap:
        defaultMode: 420
        name: ca-file
        items:
          - key: service-ca.crt
            path: tls-ca-bundle.pem
      name: trusted-ca
EOF
```

```bash
oc create -f pod-with-ca.yml
```

可以发现，直接访问成功，不需要-k跳过证书验证了

```bash
sh-4.4$ curl https://hello.default.svc
<html>
  <body>
    <h1>Hello, world from nginx!</h1>
  </body>
</html>
```

研究一下证书交互过程

```text
sh-4.4$ curl https://hello.default.svc -vv -I
* Rebuilt URL to: https://hello.default.svc/
*   Trying 172.30.246.118...
* TCP_NODELAY set
* Connected to hello.default.svc (172.30.246.118) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/pki/tls/certs/ca-bundle.crt
  CApath: none
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, [no content] (0):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=hello.default.svc
*  start date: Dec 20 08:36:30 2024 GMT
*  expire date: Dec 20 08:36:31 2026 GMT
*  subjectAltName: host "hello.default.svc" matched cert's "hello.default.svc"
*  issuer: CN=openshift-service-serving-signer@1706011744
*  SSL certificate verify ok.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* TLSv1.3 (OUT), TLS app data, [no content] (0):
* TLSv1.3 (OUT), TLS app data, [no content] (0):
* TLSv1.3 (OUT), TLS app data, [no content] (0):
* Using Stream ID: 1 (easy handle 0x561a9eab7a00)
* TLSv1.3 (OUT), TLS app data, [no content] (0):
> HEAD / HTTP/2
> Host: hello.default.svc
> User-Agent: curl/7.61.1
> Accept: */*
>
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, [no content] (0):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS app data, [no content] (0):
* Connection state changed (MAX_CONCURRENT_STREAMS == 128)!
* TLSv1.3 (OUT), TLS app data, [no content] (0):
* TLSv1.3 (IN), TLS app data, [no content] (0):
* TLSv1.3 (IN), TLS app data, [no content] (0):
< HTTP/2 200
HTTP/2 200
< server: nginx/1.14.1
server: nginx/1.14.1
< date: Fri, 20 Dec 2024 08:52:26 GMT
date: Fri, 20 Dec 2024 08:52:26 GMT
< content-type: text/html
content-type: text/html
< content-length: 72
content-length: 72
< last-modified: Wed, 26 Jun 2019 22:19:37 GMT
last-modified: Wed, 26 Jun 2019 22:19:37 GMT
< etag: "5d13ef79-48"
etag: "5d13ef79-48"
< accept-ranges: bytes
accept-ranges: bytes

<
* Connection #0 to host hello.default.svc left intact
```

也可以用下面的方法验证过程

```bash
openssl s_client -connect hello.default.svc:443
```
