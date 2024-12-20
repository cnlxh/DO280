```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 公开非 HTTP/SNI 应用

# 课程目标

- 通过使用负载平衡器服务，将应用公开给外部访问。​

- 使用辅助网络公开应用供外部访问。​

# 负载平衡器服务

ingress和route提供了一种方式来公开http类的工作负载。​但是，在某些情况下，ingress和route不足以公开容器集提供的服务。还有很多非http类型的，最起码来说，比如我们ssh协议的22号端口，就不能用ingress来做，针对此类的服务，我们用k8s中的Service来实现，至于说<mark>对外的统一访问，我们用loadbalance的服务类型</mark>即可

对于将k8s的service设置为loadbalance类型，需要配合外部的负载均衡器实现，类似公司里的F5设备、或者公有云中的各种负载均衡服务都可以，不顾哦在我们的课程中，我们没有这些，所以课程中部署了metallb在裸金属集群中提供负载均衡服务，说到这里，我们来介绍一下K8S中几种常见的服务类型

## k8S中的服务类型

**内部沟通**

纯集群内部的沟通，ClusterIP 类型的服务在集群内提供服务访问。对外没有任何路由

**外部沟通**

NodePort 和 LoadBalancer 类型的服务常用于对外沟通，这些类型会将集群中运行的服务向集群外部公开。

NodePort工作在30000-32767的端口范围

## 负载均衡服务案例

在使用服务之前，我们需要先部署一个后端，不然前端接受请求后，不知道发给谁

由于http类型的服务部署简单，我们就用nginx演示，你也可以做做教材上的视频摄像头练习

```yaml
cat > deployment-service.yml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lxh-pod-backend
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
        image: registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
EOF
```

```bash
oc create -f deployment-service.yml
```

部署一个type: LoadBalancer的服务，此服务工作在80端口，80端口收到请求时，会转发给具有app: nginx标签的pod中的8080端口

```yaml
cat > loadbalancer.yml <<-EOF
apiVersion: v1
kind: Service
metadata:
  name: loadbalance-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
EOF
```

在我们的环境中，metallb默认提供了192.168.50网段的外部IP

```bash
[student@workstation ~]$ oc create -f loadbalancer.yml
service/loadbalance-service created
[student@workstation ~]$ oc get -f loadbalancer.yml
NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
loadbalance-service   LoadBalancer   172.30.214.47   192.168.50.20   80:30140/TCP   3s
```

我们来试试从负载均衡IP发起请求

```bash
[student@workstation ~]$ curl 192.168.50.20
<html>
  <body>
    <h1>Hello, world from nginx!</h1>
  </body>
</html>
```

# Multus 辅助网络

Kubernetes 管理容器集网络和服务网络。​容器集网络为每个容器集提供网络接口，在某些情况下，将一些<mark>容器集连接到其他网络</mark>可以带来好处或帮助满足需求。​例如，使用具有专用资源的专用网络可以提高特定流量的性能。​此外，专用网络可以具有与默认网络不同的安全属性，有助于满足安全性要求。​

<mark>Multus CNI</mark>（容器网络接口）插件有助于<mark>将容器集附加到自定义网络</mark>。​这些自定义网络可以是集群外部的现有网络，也可以是集群内部的自定义网络。​

## Multus 辅助网络案例

### 确认网络环境现状

在我们的课程环境中，master01这个节点上既是控制面也是数据面，所有的工作负载都运行在此，我们看看它的网络接口

ens4 接口是额外的网络接口，可用于需要额外网络的练习。此接口连接到 192.168.51.0/24 网络，其 IP 地址为 192.168.51.10

```bash
[student@workstation ~]$ oc debug node/master01 -- chroot /host ip addr
Temporary namespace openshift-debug-d5mzc is created for debugging node...
Starting pod/master01-debug-s9fgk ...
To use host binaries, run `chroot /host`
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master ovs-system state UP group default qlen 1000
    link/ether 52:54:00:00:32:0a brd ff:ff:ff:ff:ff:ff
    altname enp0s3
3: ens4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:01:33:0a brd ff:ff:ff:ff:ff:ff
    altname enp0s4
    inet 192.168.51.10/24 brd 192.168.51.255 scope global dynamic noprefixroute ens4
       valid_lft 412787654sec preferred_lft 412787654sec
    inet6 fe80::878:11eb:73df:8a1b/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

和master01有路由的机器是utility这台，只有这台才能ping通ens4的ip，而workstation是不行的，我们来试试

```bash
[student@workstation ~]$ ping -c 1 192.168.51.10
PING 192.168.51.10 (192.168.51.10) 56(84) bytes of data.

--- 192.168.51.10 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms

[student@workstation ~]$ ssh root@utility
[root@utility ~]# ping -c 1 192.168.51.10
PING 192.168.51.10 (192.168.51.10) 56(84) bytes of data.
64 bytes from 192.168.51.10: icmp_seq=1 ttl=64 time=0.683 ms

--- 192.168.51.10 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.683/0.683/0.683/0.000 ms
```

网络验证好之后，我们要知道，稍后我们在集群的pod中，添加的额外接口，只能在utility这台机器上才能访问和ping通

### 向集群发布业务

```bash
```yaml
cat > deployment-service.yml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multus-test
  labels:
    app: nginx
spec:
  replicas: 1
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
        image: registry.ocp4.example.com:8443/redhattraining/hello-world-nginx:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
EOF
```

```bash
oc create -f deployment-service.yml
```

确认服务工作正常

```bash
[student@workstation ~]$ oc get pod -o wide
multus-test-6645d8bb58-mgrfn       1/1     Running   0          2m33s   10.8.0.155   master01   <none>           <none>
```

### 向集群发布辅助网络

这里我们发布了一个名为custom的网络，这个网络和主机上的ens4接口关联，并对外提供192.168.51.10/24

```yaml
cat > multus-network.yml <<-'EOF'
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: custom
spec:
  config: |-
    {
      "cniVersion": "0.3.1",
      "name": "custom",
      "type": "host-device",
      "device": "ens4",
      "ipam": {
        "type": "static",
        "addresses": [
          {"address": "192.168.51.10/24"}
        ]
      }
    }
EOF
```

```bash
[student@workstation ~]$ oc create -f multus-network.yml
networkattachmentdefinition.k8s.cni.cncf.io/custom created
[student@workstation ~]$ oc get -f multus-network.yml
NAME     AGE
custom   3s
[student@workstation ~]$ oc describe -f multus-network.yml
Name:         custom
Namespace:    laoli
Labels:       <none>
Annotations:  <none>
API Version:  k8s.cni.cncf.io/v1
Kind:         NetworkAttachmentDefinition
Metadata:
  Creation Timestamp:  2024-12-20T12:11:08Z
  Generation:          1
  Resource Version:    262311
  UID:                 16b47917-5729-4e2e-b5a6-ca98a0227969
Spec:
  Config:  {
  "cniVersion": "0.3.1",
  "name": "custom",
  "type": "host-device",
  "device": "ens4",
  "ipam": {
    "type": "static",
    "addresses": [
      {"address": "192.168.51.10/24"}
    ]
  }
}
Events:  <none>
```

### 更新业务pod添加辅助网络

写一个补丁，用于添加我们的辅助网络

```yaml
cat > multus-patch.yaml <<-EOF
spec:
  template:
    metadata:
      annotations:
        k8s.v1.cni.cncf.io/networks: custom
EOF
```

更新我们的业务pod

```bash
oc patch deployment multus-test --patch-file multus-patch.yaml
```

### 确认业务pod已经拥有辅助网络

很好，我们看到pod已经有了net1这个网卡，并拥有192.168.51.10这个地址

```yaml
[student@workstation ~]$ oc get pod multus-test-66858889c-9jvh7 -o yaml | grep -B 20 custom
apiVersion: v1
kind: Pod
metadata:
  annotations:
    k8s.ovn.org/pod-networks: '{"default":{"ip_addresses":["10.8.0.157/23"],"mac_address":"0a:58:0a:08:00:9d","gateway_ips":["10.8.0.1"],"routes":[{"dest":"10.8.0.0/14","nextHop":"10.8.0.1"},{"dest":"172.30.0.0/16","nextHop":"10.8.0.1"},{"dest":"100.64.0.0/16","nextHop":"10.8.0.1"}],"ip_address":"10.8.0.157/23","gateway_ip":"10.8.0.1"}}'
    k8s.v1.cni.cncf.io/network-status: |-
      [{
          "name": "ovn-kubernetes",
          "interface": "eth0",
          "ips": [
              "10.8.0.157"
          ],
          "mac": "0a:58:0a:08:00:9d",
          "default": true,
          "dns": {}
      },{
          "name": "laoli/custom",
          "interface": "net1",
          "ips": [
              "192.168.51.10"
          ],
          "mac": "52:54:00:01:33:0a",
          "dns": {}
      }]
    k8s.v1.cni.cncf.io/networks: custom
```

### 通过辅助网络访问业务

```bash
[root@utility ~]# ping -c 1 192.168.51.10
PING 192.168.51.10 (192.168.51.10) 56(84) bytes of data.
64 bytes from 192.168.51.10: icmp_seq=1 ttl=64 time=0.872 ms

--- 192.168.51.10 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.872/0.872/0.872/0.000 ms
[root@utility ~]# curl 192.168.51.10:8080
<html>
  <body>
    <h1>Hello, world from nginx!</h1>
  </body>
</html>
```


