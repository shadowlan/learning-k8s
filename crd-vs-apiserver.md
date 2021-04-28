# CRD vs apiserver
自定义资源[Custom Resource Definition]和apiserver是[多种扩展K8s的方式](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/)中的其中两种。
究竟是利用CRD+Controller的Operator模式来实现声明式API还是通过k8s聚合层来聚合独立运行的apiserver，
取决于应用的实际情况，[官方文档](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#should-i-add-a-custom-resource-to-my-kubernetes-cluster)给出了基础的比较。

## CR/CRD

`CRD + Controller = Operator`

要理解什么是Custom Resource，需要新理解Resource是什么，在k8s中，Resource是k8s API的一个endpoint，
存储某个类型的API对象的集合，比如内置的pods resource包含一组Pod对象。而Custom Resource则是对k8s API的
扩展，在默认k8s安装中不一定可用。[这篇文章](https://itnext.io/crd-is-just-a-table-in-kubernetes-13e15367bbe4)拿CR和CRD与数据库做类比，
把CRD比做数据库中的表，而CR则是表里的每条记录。CRD的主要目的是让 kube-apiserver 能够认识新的对象类别（Kind).

Controller是Kubernetes以及任何Operator的核心。Controller的职责是确保对于任何给定的对象，其实际状态
（包括群集状态，以及潜在的外部状态，例如正在运行的Kubelet容器或云提供商的负载平衡器）与对象中的所需状态相匹配。
每个Controller专注于一个root Kind，但可以与其他Kind进行交互。在Controller runtime中，实现特定Kind协调的逻辑称为Reconciler。
Reconciler获取对象的名称，并返回是否需要重试。

用户在声明了一个CRD后，可以通过kubectl创建新的CR，但是如果后端没有Controller来处理CR的增删改，那么新建的CR就如同静态表一样，
不会起到任何实作用。当然极端案例是用户的确只创建了CRD和CR，通过K8s API的[GVK(Group/Version/Kind)](https://book.kubebuilder.io/cronjob-tutorial/gvks.html) endpoint直接访问CR，把k8s当做一个数据库来做CR增删改，这种简单的情况因为CRD后端没有与之关联的应用，就像通过SQL给数据库中某个
孤立的表增删改一行记录一样，没有对任何应用发生作用。

为了实现可用CRD + Controller，可以使用[kubebuilder](https://book.kubebuilder.io/)来快速构建，
至于如何使用kubebuilder，会在另一篇文章中介绍。

## 独立apiserver

由于CRD无法实现的以下操作：
  - 使用户的apiserver采用不同的存储API而不是ETCDv3
  - 将长期运行的子资源/端点（例如websocket）扩展为您自己的资源
  - 将用户的apiserver与其他任何外部系统集成
  
另外有些用户自定义API不符合声明式API，无法利用CRD来实现，或者不需要kubectl支持，那么可以开发独立的apiserver。  
在开发独立apiserver之前，需要保证配置好k8s的[聚合层](https://kubernetes.io/zh/docs/tasks/extend-kubernetes/configure-aggregation-layer/)。
聚合层在kube-apiserver进程内运行。在扩展资源注册之前，聚合层不做任何事情。 要注册API，用户必须添加一个APIService对象，
用它来“申领” Kubernetes API中的URL路径。 这以后，聚合层将会把发给该API路径的所有内容（例如/apis/myextension.mycompany.io/v1/…） 
转发到已注册的APIService。

在k8s基础上运行的独立apiserver需要符合一些规范，可以参考k8s仓库里提供的[apiserver-builder](https://github.com/kubernetes-sigs/apiserver-builder-alpha)，
该工具集旨在便于用户快速开始开发一个自定义apiserver。
```
Apiserver Builder是使用Kubernetes apiserver聚合来构建原生Kubernetes扩展的库和工具的集合。
聚合的apiserver使用户能够自定义apiserver，以执行CR[D]
apiserver-builder提供了库，代码生成器和工具，使用户可以在一个下午构建并运行基本的apiserver，
同时提供所有的钩子，以便在从头开始构建时提供相同的功能。
```

另外，用户也可以利用k8s提供的[apiserver](https://github.com/kubernetes/apiserver)库来实现自定义apiserver。

### 注册APIService对象示例

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: <注释对象名称>
spec:
  group: <扩展 Apiserver 的 API 组名>
  version: <扩展 Apiserver 的 API 版本>
  groupPriorityMinimum: <APIService 对应组的优先级, 参考 API 文档>
  versionPriority: <版本在组中的优先排序, 参考 API 文档>
  service:
    namespace: <拓展 Apiserver 服务的名字空间>
    name: <拓展 Apiserver 服务的名称>
  caBundle: <PEM 编码的 CA 证书，用于对 Webhook 服务器的证书签名>
```

```yaml
# antrea system group APIService 实例。
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  labels:
    app: antrea
  name: v1beta1.system.antrea.io
spec:
  group: system.antrea.io
  groupPriorityMinimum: 100
  service:
    name: antrea
    namespace: kube-system
  version: v1beta1
  versionPriority: 100
```
