#   kubernetes安装(ubuntu20.04)

### 1.先安装docker: 

```sh
$ curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
$ add-apt-repository "deb https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
$ apt update && apt install -y docker-ce 
$ docker run hello-world
```

### 2.配置k8s的更新源，推荐阿里云: 

```sh
$ echo "deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
#这里先运行一下 apt udpate, 会报错，原因是缺少相应的key，可以通过下面的命令添加(BA07F4FB 为上面报错的key后8位)
$ sudo gpg --keyserver keyserver.ubuntu.com --recv-keys BA07F4FB #对安装包进行签名
$ sudo gpg --export --armor BA07F4FB | sudo apt-key add -
$ sudo apt-get update
```

### 3. 关闭虚拟内存 

```sh
$ sudo swapoff -a #暂时关闭
$ sudo vim /etc/fstab #永久关闭，注释掉swap那一行，推荐永久关闭
```

### 4.安装k8s

```sh
$ sudo apt-get update && apt-get install -y kubelet kubeadm kubectl
```

### 5.配置Master节点的k8s，并使用 kubeadm 拉取镜像

```sh
1.修改IP地址为master节点的IP地址并配置pod地址
#192.168.3.130 为master主机上的ip
kubeadm init \
--apiserver-advertise-address=192.168.3.130 \
--image-repository registry.aliyuncs.com/google_containers  \
--pod-network-cidr=10.244.0.0/16 
#运行后的输出如下
Your Kubernetes control-plane has initialized successfully!
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.3.130:6443 --token 0tkm30.zk7pw40asvr9j577 \
    --discovery-token-ca-cert-hash sha256:1a8ee4d338f9339bb70ced6c067bcae8470f527d21ffa8425a3addd006040a5a 
    
2.输入以下命令:
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 6.其中`kubeadm`用于初始化环境，`kubectl`用于操作`kubelet`。 设置开机启动 

```sh
#开机启动docker
$ sudo systemctl enable docker && systemctl start docker
#开机启动kubelet
$ sudo systemctl enable kubelet && systemctl start kubelet
```

### 7.查看启动状况

```sh
$ kubectl get nodes
NAME      STATUS     ROLES                  AGE    VERSION
rd3g1-1   NotReady   control-plane,master   8m2s   v1.20.1
$ kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE          ERROR
controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused   
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused   
etcd-0               Healthy     {"health":"true"}  
```

### 8.添加网络插件（使用weave）

```sh
$ sysctl net.bridge.bridge-nf-call-iptables=1
$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

### 9.配置Node节点

```sh
$ kubeadm join 192.168.3.130:6443 --token 0tkm30.zk7pw40asvr9j577 \
    --discovery-token-ca-cert-hash sha256:1a8ee4d338f9339bb70ced6c067bcae8470f527d21ffa8425a3addd006040a5a 
```

### 10.部署ningx应用，测试集群

**(1)在Kubernetes集群中创建一个pod，验证是否正常运行：** 

```sh
$ kubectl create deployment nginx --image=nginx
deployment.apps/nginx created
$ kubectl expose deployment nginx --port=80 --type=NodePort
service/nginx exposed
$ kubectl get pod,svc
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-6799fc88d8-4mqtl   1/1     Running   0          7m11s

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        3h14m
service/nginx        NodePort    10.107.107.200   <none>        80:32342/TCP   6m48s

```

**(2)部署成功：** 

```sh
$ curl 127.0.0.1:32342
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

**(3)快速扩容为3副本：** 

```sh
$ kubectl scale deployment nginx --replicas=3
deployment.apps/nginx scaled
$ kubectl get pod,svc
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-6799fc88d8-4mqtl   1/1     Running   0          9m7s
pod/nginx-6799fc88d8-b62t9   1/1     Running   0          16s
pod/nginx-6799fc88d8-s2h78   1/1     Running   0          16s

NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        3h15m
service/nginx        NodePort    10.107.107.200   <none>        80:32342/TCP   8m44s

```

