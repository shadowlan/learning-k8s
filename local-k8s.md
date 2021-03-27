# 本地k8s搭建

又是忙碌紧凑的一天，虽然已经用了k8s好几年了，发现除了N早用的时候手动搭建过之外，已经很久没有在本地搭建过环境了。今天尝试通过kubeadm在本地搭建一个k8s环境。期间遇到了不少坑，记录在此。

## 准备虚机

通过Vmware Fusion创建了三个Ubuntu20.04（ubuntu-20.04.2-live-server-amd64.iso）版本的linux机器。iso是从[这里](http://mirrors.aliyun.com/ubuntu-releases/20.04/)下载的。
通过Fusion UI配置虚机，在安装过程中除了选择装openssh，其他都没选，机器起来的时候发现连基本的ifconfig都没有😳，于是又通过提示安装了net-tools:`apt install net-tools`。
中间配置重启几次发现机器的IP会变，于是给三台机器配置了静态IP。具体配置步骤如下：

1. 修改/etc/netplan/00-installer-config.yaml文件，内容如下
```
network:
  ethernets:
    ens33:
      dhcp4: false
      addresses: [172.16.6.128/24]
      optional: true
      gateway4: 172.16.6.2
      nameservers:
        addresses: [172.16.6.2,114.114.114.114,8.8.8.8] 
  version: 2
  ```
2. 使配置生效：`sudo netplan apply`

三台机器hostname和ip信息如下：

| hostname     | ip           |
| ------------- | ------------ |
| master        | 172.16.6.128 |
| node1         | 172.16.6.133 |
| node2         | 172.16.6.134 |


## 安装docker

比较简单，一行搞定`sudo apt-get update;sudo apt install docker.io`

## 配置安装kubeadm

安装kubeadm前需要做一些环境检查和准备，我没有完全按照[官方文档](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)来，官方文档应该是最全的，

1. 确认br_netfilter模块是否加载`smod | grep br_netfilter`,如果没加载运行`sudo modprobe br_netfilter`
2. [配置systemd为docker的cgroup driver](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker)
```bash
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```
3. 关闭swap
   1. 临时： sudo​ swapoff -a （临时）
   2. 永久： `vi /etc/fstab`, 注释掉swap行，并reboot  
tips: 通过free -m 或者cat /proc/swaps 查看swap是否为0 或者空来确认swap已禁用。

4. 安装kubernetes repo的签名公钥`curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -`
5. 添加k8s的apt源
```
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
我本地访问官方源太慢，最后用国内的源替代上面deb行内的url： `http://mirrors.ustc.edu.cn/kubernetes/apt`来安装后续工具。  
另外发现http://mirrors.ustc.edu.cn真是个宝藏，里面有非常多各类系统的镜像源。

6. 安装kubeadm及相关工具
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## kubeadm安装cluster

可以运行`kubeadm -h`对其使用有个基本了解。主要是在master节点启动`kubeadm init ...`控制面板相关组件。在node节点启动`kubeadm join ...`加入到cluster中。这里记录了启动一个新cluster时遇到的一些坑。

1. 在master节点运行`sudo kubeadm init`。

问题1： 遇到的第一坑是镜像无法下载，因为kubeadm默认使用k8s官方镜像源 k8s.gcr.io，国内访问超时。  
Fix: 在网上找到国内可用镜像源，速度还是非常快的，下面记录如何替换： 
a. 通过`sudo kubeadm config print init-defaults > kubeadm.conf`导出默认配置文件。
b. 查看kubeadm使用的镜像版本： `sudo kubeadm config images list`,我本地输出是v1.20.5  
c. 将kubeadm.conf文件内的如下内容替换：  
```
imageRepository: k8s.gcr.io -> imageRepository: registry.aliyuncs.com/google_containers
kubernetesVersion: v1.20.0 -> kubernetesVersion: v1.20.5
```
！！注意： k8s的network addon安装是在kubeadm init创建cluster之后运行，如果选择的network addon对运行kubeadm init时有参数要求，需要在这里加上。比如antrea需要传`--pod-network-cidr=<CIDR Range for Pods>`来启用NodeIpamController。  
另外可以参考[官方文档](https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file)了解更多如何使用配置文件启动kubeadm init。

修改配置后可以用`sudo kubeadm config images pull --config kubeadm.conf`来拉取镜像，然后执行`sudo kubeadm init --config kubeadm.conf`.但是再次遇到问题...

问题2: kubelet始终无法正常启动，etcd，api-server等启动后退出，`kubeadm init`失败。  
`kubeadm init`失败后会有些提示，让做一些基本检查，我这里看到的问题是查看了etcd的容器log后发现etcd无法正常启动，错误显示`etcdmain: listen tcp 1.2.3.4:2380: bind: cannot assign requested address`.  
Fix: 将kubeadm.conf文件内的如下内容替换：  
```
advertiseAddress: 1.2.3.4 -> advertiseAddress: 0.0.0.0
```

运行`sudo kubeadm reset`以便清理上一步产生的容器，配置文件等。
再次运行`sudo kubeadm init --config kubeadm.conf`,成功后会提示在node节点安装network addon并运行`kubectl join ...`。

2. 安装network addon  
根据kubeadm init的最终成功的output提示，在本地配置kubeconfig文件。
```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

然后运行`kubectl apply -f 'path of your network-add-on yaml file'`,我这里安装的是antrea，所以运行的是`kubectl apply -f https://raw.githubusercontent.com/vmware-tanzu/antrea/main/build/yamls/antrea.yml`.

3. 在node节点运行`kubeadm join ...`

这一步比较简单，直接拷贝上一步`kubeadm init`执行成功后输出的`kubectl join`那一行，然后在node1和node2上执行即可。

4. 修改镜像tag  
虽然通过kubeadm.conf修改了镜像库为 'registry.aliyuncs.com/google_containers'， 但是考虑到很多开源应用的helm chart肯定指向的是k8s.gcr.io源，所以还是将三台机器上的镜像都修改了一下。
```
sudo docker tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.20.5 k8s.gcr.io/kube-apiserver:v1.20.5
sudo docker tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.20.5  k8s.gcr.io/kube-controller-manager:v1.20.5
sudo docker tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.20.5  k8s.gcr.io/kube-scheduler:v1.20.5
sudo docker tag registry.aliyuncs.com/google_containers/kube-proxy:v1.20.5  k8s.gcr.io/kube-proxy:v1.20.5
sudo docker tag registry.aliyuncs.com/google_containers/pause:3.2  k8s.gcr.io/pause:3.2
sudo docker tag registry.aliyuncs.com/google_containers/etcd:3.4.13-0  k8s.gcr.io/etcd:3.4.13-0
sudo docker tag registry.aliyuncs.com/google_containers/coredns:1.7.0  k8s.gcr.io/coredns:1.7.0
```