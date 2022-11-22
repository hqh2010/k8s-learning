# k8s集群安装教程

1. 切换到root用户

    ```bash
    sudo su
    ```

2. 配置host

    ```bash
    # 添加节点信息，如果只有一个节点，则只用加一个配置
    
    192.169.1.210 k8s-master
    192.169.1.211 k8s-node1
    192.169.1.212 k8s-node2
    
    # 下面的第2行为新增的配置信息
    uos@k8s-master:~/Desktop$ cat -n /etc/hosts
         1  127.0.0.1       localhost
         2  192.168.122.217 k8s-master
         3  # The following lines are desirable for IPv6 capable hosts
         4  ::1     localhost ip6-localhost ip6-loopback
         5  fe00::0 ip6-localnet
         6  ff00::0 ip6-mcastprefix
         7  ff02::1 ip6-allnodes
         8  ff02::2 ip6-allrouters
    ```

3. 修改主机名
    每台主机都要修改

   ```bash
   # 如果只有一个节点，则只用改一个
   hostnamectl set-hostname k8s-master
   hostnamectl set-hostname k8s-node1
   hostnamectl set-hostname k8s-node2
   ```

4. 关闭swap分区
    每台主机都要关闭

   ```bash
   # 临时关闭
   swapoff -a
   # 永久关闭(老版本)
   sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab
   # 永久关闭(新版本)
   只需要编辑 /etc/fstab 文件，不要注释掉 swap 那一行
   而是在其后面默认的挂载选项 defaults 后面加上 noauto 变成 defaults,noauto。
   
   # 查看是否注释
   cat /etc/fstab
   # 注释后的配置如下：
   # /dev/vda4
   UUID=fd0bd2ac-03f9-4bf0-9931-f21db25e0e4d       none            swap            defaults,noauto,pri=-2  0 0
   
   ```

   * [https://www.freedesktop.org/software/systemd/man/systemd.swap.html](https://www.freedesktop.org/software/systemd/man/systemd.swap.html)
   * [openSUSE Tumbleweed 中禁用 SWAP](https://cnzhx.net/blog/disable-swap-in-opensuse-tumbleweed/)

5. 安装docker

    * [参考清华Docker Community Edition 镜像使用帮助安装docker](https://mirrors.tuna.tsinghua.edu.cn/help/docker-ce/)

6. 修改docker Cgroup驱动

    ```bash
    # 给docker添加如下配置
    {
    "exec-opts":["native.cgroupdriver=systemd"]
    }
    # 配置后的结果如下：
    uos@k8s-master:~/Desktop$ cat /etc/docker/daemon.json
    {
    "registry-mirrors": [
    "https://kfwkfulq.mirror.aliyuncs.com",
    "https://2lqq34jg.mirror.aliyuncs.com",
    "https://pee6w651.mirror.aliyuncs.com",
    "https://registry.docker-cn.com",
    "http://hub-mirror.c.163.com"
    ],
    "exec-opts":["native.cgroupdriver=systemd"]
    }
    ```

7. 安装k8s

    * [添加阿里云 Kubernetes 镜像仓库](https://developer.aliyun.com/mirror/kubernetes?spm=a2c6h.13651102.0.0.73bf1b11EI1d2X)

    * [【Kubernetes系列】K8s由1.24.1降级为1.23.8
](https://blog.csdn.net/u012069313/article/details/125561711)

    * [Kubernetes 1.24 将结束对dockershim 的支持
](https://www.51cto.com/article/707507.html)

    * [Kubernetes 1.24 的删除和弃用](https://kubernetes.io/zh-cn/blog/2022/04/07/upcoming-changes-in-kubernetes-1-24/)

    ```bash
    apt-get install -y kubelet=1.23.8-00 kubeadm=1.23.8-00 kubectl=1.23.8-00
    ```

8. 设置kubectl命令补全

    ```bash
    apt-get install bash-completion
    ```

    ```bash
    echo 'source /usr/share/bash-completion/bash_completion' >>~/.bashrc
    ```

    ```bash
    echo 'source <(kubectl completion bash)' >>~/.bashrc
    ```

9. 初始化k8s-master节点

    ```bash
    # 192.168.2.248 要换成k8s-master的IP地址
    kubeadm init \
    --apiserver-advertise-address 192.168.2.248
    --image-repository registry.aliyuncs.com/google_containers \
    --pod-network-cidr=10.244.0.0/16 \
    --service-cidr=10.1.0.0/16 
    ```
    
10. 拷贝 kubectl 使用的连接 k8s 认证文件到默认路径

    ```bash
    exit
    ```

    ```bash
    mkdir -p $HOME/.kube
    ```

    ```bash
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    ```

    ```bash
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

    ``` bash
    echo 'export KUBECONFIG=$HOME/.kube/config' >>~/.bashrc
    ```

    ```bash
    # 查询master节点信息，如果部署多个节点的集群，需要在节点电脑执行下面生成的kubeadm join命令
    kubeadm token list
    root@k8s-master:~# kubeadm token create --print-join-command
    kubeadm join 192.168.122.217:6443 --token 5ek7s0.a5hyaxdmje9768xq --discovery-token-ca-cert-hash sha256:98e6dee99c9df129b8bcbfa189c07f0649ac56a6ca6b5a1a5c0d30f6ec58f84f
    ```

11. 安装Pod网络插件 calico

    ```bash
    kubectl apply -f https://docs.projectcalico.org/v3.18/manifests/calico.yaml
    ```

    * [k8s master节点状态为 NotReady问题解决
](https://blog.csdn.net/w849593893/article/details/119883531)

12. 添加node节点
    * 以下操作在Node节点上执行

    ```bash
    sudo su
    ```

    ```bash
    kubeadm join 192.168.122.217:6443 --token 5ek7s0.a5hyaxdmje9768xq --discovery-token-ca-cert-hash sha256:98e6dee99c9df129b8bcbfa189c07f0649ac56a6ca6b5a1a5c0d30f6ec58f84f
    ```
    
    * [kubernetes忘记token或者token过期怎么加入k8s集群](https://www.cnblogs.com/linyouyi/p/10850904.html)
    
13. k8s dashboard
    * [部署和访问 Kubernetes 仪表板（Dashboard）
    ](https://kubernetes.io/zh-cn/docs/tasks/access-application-cluster/web-ui-dashboard/)
        1. 部署 Dashboard UI

        ```bash
        kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.0/aio/deploy/recommended.yaml
        ```

    * [https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/README.md](https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/README.md)

    * [https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md)

        2. 更改ClusterIP为NodePort

        ```bash
        kubectl -n kubernetes-dashboard edit service kubernetes-dashboard
        ```

        3. 创建配置文件 dashboard-adminuser.yaml

        ```plain
        apiVersion: v1
        kind: ServiceAccount
        metadata:
        name: admin-user
        namespace: kubernetes-dashboard
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
        name: admin-user
        roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: ClusterRole
        name: cluster-admin
        subjects:
        - kind: ServiceAccount
        name: admin-user
        namespace: kubernetes-dashboard
        ```

        4. 执行

        ```bash
        kubectl apply -f dashboard-adminuser.yaml
        ```

        5. k8s忘记dashboard密码

        ```bash
        kubectl -n kubernetes-dashboard get secret
        ```

        ```bash
        kubectl describe secret admin-user-token-76rq7 -n kubernetes-dashboard
        ```

        6. 获取 kubernetes-dashboard 对外暴露端口

        ```bash
        kubectl get svc --all-namespaces
        ```

        7. 访问k8s dashboard
        ```bash
        https://192.169.1.210:32188/#/login
        ```
        8. [Chrome您的连接不是私密连接解决办法
        ](https://www.jianshu.com/p/1719a27137e3)
        ```plain
        thisisunsafe
        ```
14. 查询安装状态

查询节点应用安装情况

```bash
sudo kubectl get pods -A
```

查询节点信息，需要在master查询

```bash
sudo kubectl get nodes
```
描述一个pod

```bash
kubectl describe pods/nginx
```

描述pod.json中的资源类型和名称指定的pod

```bash
kubectl describe -f pod.json
```

描述所有的pod

```bash
kubectl describe pods
```

15. 参考资料

    * [云原生Java架构师的第一课K8s+Docker+KubeSphere+DevOps
](https://www.bilibili.com/video/BV13Q4y1C7hS?p=31&vd_source=2878cbece52fc3431df4c7c4b77a8884)

https://kubernetes.io/zh-cn/docs/concepts/overview/components/
