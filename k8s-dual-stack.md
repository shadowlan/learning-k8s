# k8s dual-stack

用kubeadm init安装时，默认在local k8s cluster上只启用ipv4，如果想同时支持ipv4和ipv6,需要做一些额外配置，这里记录下具体的步骤和配置。

## 启用ipv6转发

`sudo sysctl -w net.ipv6.conf.all.forwarding=1`

## 配置文件启动kubeadm init

下面是启用ipv6所需要的基本配置文件，要支持ipv6,重点是要将featureGates里的IPv6DualStack设置为true，另外需要将advertiseAddress的ip配置为master节点的IP。serviceSubnet和podSubnet注意不要和node节点所在子网有重合。
在master节点上创建文件kubeadm.conf,然后拷贝下面的内容：

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
nodeRegistration:
  name: "master"
localAPIEndpoint:
  advertiseAddress: "<ip-of-your-master-node>"
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
featureGates:
  IPv6DualStack: true
networking:
  serviceSubnet: "172.19.0.0/16,fd74:ca9b:0172:0019::/110"
  podSubnet: "172.18.0.0/16,fd74:ca9b:0172:0018::/64"
apiServer:
  certSANs:
  - "<ip-of-your-master-node>"
kubernetesVersion: v1.19.2
```

运行`kubeadm init --config kubeadm.conf`.

注意：在配置文件里提供的kubernetesVersion，我这里是v1.19.2，master/node机器上安装的kubelet版本需要和这个一致，否则运行init时会有提示。因为之前我在本地机器安装的版本是1.20.5，导致需要重新安装，这里把命令记录下。

```bash
# on ubuntu 20.04
# check kubelet version
apt list kubelet
# check all available kubelet version
apt list kubelet -a
# install specific version
apt-get install kubelet=v1.19.2-00
```

## 参考文章

- [Dual Stack IPv4/IPv6 on Kubernetes](https://medium.com/@prashy.pk/dual-stack-ipv4-ipv6-on-kubernetes-4a68c7f985b3)
- [Kubernetes by kubeadm config yamls](https://medium.com/@kosta709/kubernetes-by-kubeadm-config-yamls-94e2ee11244)
