```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 身份验证和授权

# 课程目标

- 允许并保护 OpenShift 集群内应用的网络连接。​

- 限制项目和 pod 之间的网络流量。​

- 配置和使用自动服务证书。​

# 利用 TLS 保护外部流量

OpenShift 容器平台提供了多种方式向外部网络公开您的应用。​您可以公开 HTTP 和 HTTPS 流量、​TCP 应用以及非 TCP 流量。​其中一些方法是服务​**type**，如 `NodePort` 或负载平衡器，而另一些则使用自己的 API 资源，如 `Ingress` 和 `Route`。​

借助 OpenShift 路由，您可以向外部网络公开您的应用，从而通过可公开访问的唯一主机名称访问应用。​路由依赖于路由器插件，将来自公共 IP 的流量重定向到 pod。​

下图显示了路由如何公开在集群中作为 pod 运行的应用：

![](https://gitee.com/cnlxh/do280/raw/master/images/chapter4/network-sdn-routes-network.svg)



**保护route的方案一般有三种，注意用前两种**



**边缘**

使用边缘终止时，TLS 终止发生在路由器上，在流量路由到 pod 之前。​路由器服务于 TLS 证书，因此您必须将它们配置到路由中；否则，OpenShift 会将自己的证书分配到路由器，以用于 TLS 终止。​由于 TLS 是在路由器终止的，因此不会加密通过内部网络进行的、​从路由器到端点的连接。​

**直通**

借助直通终止，加密流量直接发送到目标 pod，而无需来自路由器的 TLS 终止。​在此模式中，应用负责为流量提供证书。​要支持应用和访问它的客户端之间的相互身份验证，直通是当前唯一一种方法。​

**再加密**

再加密是边缘终止的一种变体，即路由器通过证书终止 TLS，然后再加密它与端点的连接，这可能有不同的证书。​因此，完整的连接路径被加密，即使在内部网络上。​路由器使用健康检查来判断主机的真实性。​

## 边缘卸载

在边缘模式中使用路由时，客户端和路由器之间的流量会加密，但路由器和应用之间的流量则不会加密：

![](https://gitee.com/cnlxh/do280/raw/master/images/chapter4/network-sdn-routes-edge.svg)

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

![](https://gitee.com/cnlxh/do280/raw/master/images/chapter4/network-sdn-routes-passthrough.svg)

### 创建tls机密

再生成证书请求，本次申请为tls-pass.apps.ocp4.example.com

我这里服用代码，所以会覆盖前面的证书和私钥，如果需要请备份前面的

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

# 配置网络政策


