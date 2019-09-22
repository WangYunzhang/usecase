# K8S Calico Networking Inbound Connectivity on AWS

Calico是针对容器、虚拟机和基于主机工作负载的开源网络和网络安全解决方案。 Calico支持广泛的平台，包括Kubernetes，OpenShift，Docker EE，OpenStack和裸机服务。当和K8S集成时，Calico创造了为容器创造一个路由网络，使得Pod之间通讯就像正常IP连接一样，它不使用overlay，而是用纯三层的方法，使用虚拟路由器通过BGP协议传播路由到其他集群内节点和数据中心的其他部分。
关于Calico的具体原理和组件详见https://docs.projectcalico.org/，本文重点讨论AWS VPC环境下，K8S集群外部网络访问集群内Pod的方式。当然通过K8S服务IP也是一种inboud的访问方式。本文重点探讨另外一种通过路由来实现连通的方式，官方推荐的BGP Peering的方式：
 ![BGP](./BGP.png)

在数据中心网络环境下，单个worker node上运行的虚拟路由器和数据中心的路由器建立BGP邻居关系，外部网络访问集群通过该路由器到达Calico endpoint。
常见的情况是容器主机位于自己的隔离的第2层网络上，例如服务器机房中的机架或整个数据中心。 通过路由器访问该网络，路由器也是所有容器主机的默认路由器。

但是在AWS VPC环境下不像物理数据中心那样有路由器可以和容器主机建立对等连接。本文主要针对这个场景进行讨论。实例场景如下图所示：
构建一个三节点的集群，1个master节点，2个worker节点，目标是实现集群内部的pod和本VPC内的instanceA以及跨VPC的instanceB互通。
![architecture](./image.png)

## 利用kops创建K8S集群

首先我们用kops来创建K8S集群，并且启用calico CNI，详细设置过程请参考https://github.com/kubernetes/kops/blob/master/docs/aws.md，在设置好AWS环境、安装完kops、kubectl命令行工具之后，执行以下操作：

```Bash
export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id --profile kops)
export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key --profile kops)
export NAME=wyz.k8s.local
export KOPS_STATE_STORE=<s3://your-bucket-for-kops>
export ZONES=${ZONES:-"us-west-2a,us-west-2b"}

kops create cluster \
  --zones $ZONES\
  --networking calico \
  ${NAME}
--yes
```
创建成功之后的输出类似下图：
 

## 集群内部连接：
kops在AWS环境里自动创建了一个VPC1（CIDR为172.20.0.0/16），在2个AZ分别创建了一个子网（172.20.32.0/19、172.20.64.0/19）,相应地在2个子网创建了K8S集群。
然后，另外在VPC1内部新建子网172.20.96.0/19，在子网内部创建K8S集群外的instanceA，在VPC2子网subnet3内部创建instanceB，集群内成员和实例外实例instanceA的配置如下：

角色	      IP	          Subnet	      AZ	      VPC
Master	172.20.50.133  172.20.32.0/19	us-west-2a	VPC1
Worker	172.20.46.87	 172.20.32.0/19	us-west-2a	VPC1
Worker	172.20.68.184	 172.20.64.0/19	us-west-2b	VPC1
外部实例  172.20.96.124	 172.20.96.0/19	us-west-2b  VPC1

在集群内启动示例pod：
 

kops默认使用100.96.0.0/11作为pod的地址范围，四个pod分布在worker1（172.20.46.87）和woker2（172.20.68.184）上，calico在worker节点之间建立了full mesh的BGP对等连接，因此集群内部pod之间、pod和集群内的实例之间是连通的。

worker2内部内部路由表如下：
 

在woker2内可以访问woker1内的pod：
 
同样，通过在woker1内部也可以访问worker2内的pod：
 


## 同VPC内集群外部和集群内部互通
现在我们登录到instanceA，来访问pod，很显自然由于实例没有到pod地址空间的路由，无法访问pod：
 

这个时候我们可以修改instanceA所在子网的路由，增加到每个worker的pod的地址段的路由，target分别指向对应的worker节点实例：
 
关闭instanceA和worker节点的“源/目标地址检查”。
这个时候就可以从instanceA访问worker1和worker2内的pod：

 


自此我们完成了同VPC内K8S集群外部子网段和pod的网络互通。


## 利用Transit Gateway实现跨VPC访问pod

首先我们在vpc2内部创建instanceB（172.31.35.55），实例所在子网没有办法直接添加到pod的路由，应为VPC subnet的路由表里无法添加跨VPC的target，因此无法像上一步在同一个VPC里那样直接增加subnet路由来实现intanceB和pod的连通。

AWS Transit Gateway（TGW）能够将 VPC及其本地网络连接到单个网关，能够跨多个帐户和 Amazon VPC 扩展网络，在本用例中，我们可以通过TGW来转发pod的路由，具体设置方法如下：
1.创建TGW
 
2.将VPC1和VPC2分别attach到TGW
 

3.修改TGW的路由表，增加pod网段的路由，指向VPC1:
 
4.修改instanceB所在子网路由，增加pod网段的路由条目，target指向TGW：
 

5.修改K8S worker节点所在子网的路由，添加到instanceB子网的回指路由，以及到pod网段的路由：
 

6.修改worker节点实例的安全组，允许instanceB访问相应的端口：
 

7.从instanceB访问pod：
 

至此，我们完成了跨VPC inbound访问pod设置。


注意点：
不管是在从同VPC还是跨VPC的外部访问K8S集群内部pod，都需要增加子网的路由，增加路由的条目和worker节点个数相同，目前单个路由表的最大路由条数为50，虽然可以提升至最大1000，但是可能会影响网络性能，因此在实际生产环境中，需要进行相应的测试。如果是同一个子网的实例访问pod，也可以通过在实例内部操作系统层面增加路由的方式，避免撑大子网路由表。

## 参考资料：
Calico：https://docs.projectcalico.org/v3.8/introduction/
calico外部连通性：https://docs.projectcalico.org/v3.6/networking/external-connectivity
kops on aws：https://github.com/kubernetes/kops/blob/master/docs/aws.md
利用kops进行Kubernetes网络设置：https://github.com/kubernetes/kops/blob/master/docs/networking.md#calico-example-for-cni-and-network-policy
AWS Transit Gateway文档：https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html



