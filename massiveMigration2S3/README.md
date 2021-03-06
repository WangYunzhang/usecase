# 海量文件大规模实时迁移/同步到S3

## 背景：


很多WEB站点需要处理大量的图片内容，为了适应多屏的场景，往往需要对图片尺寸和质量做适应的处理，因此，特别是在电子商务应用，图片处理服务是一个非常通用的场景。
在AWS上见到一些用户使用EC和EBS来构建该服务，但是随着业务商品种类的增长，图片库会越来越膨胀（数十TB），这个时候，从成本和可架构扩展性、数据可靠性考虑，aws建议将图片storage迁移到S3.
因为S3具有99.999999999%的持久性，跨Region复制（CRR）、更低的成本，丰富的数据分层配合数据生命周期管理策略来降低成本。

数据迁移的时候用户往往有以下要求：

* 大批量小文件的迁移，速度如何保证
* 在迁移的过程中源数据在EC2上的文件系统是动态变化，迁移方法能够感知并动态同步
* 方案要能够支持A/B测试和回退



## 建议方案：

Datasync + linux NFS serve实现大批量数据迁移和同步,File gateway + lsyncd实现A/B测试阶段单个文件的实时同步
![image](./image.png)

### AWS DataSync
[AWS DataSync](https://aws.amazon.com/cn/datasync/) 是一项数据传输服务，使您能够轻松在本地存储和 Amazon S3 或 Amazon Elastic File System (Amazon EFS) 之间自动迁移数据。DataSync 可自动处理与数据传输相关的许多任务（这些任务会减慢迁移速度或增加 IT 操作负担），包括运行您自己的实例、处理加密、管理脚本、网络优化和数据完整性验证。您可以使用 DataSync 传输数据，速度最高可比开源工具快 10 倍之多。DataSync 入门非常简单：在本地部署 DataSync 代理，将其连接到文件系统或存储阵列，选择 Amazon EFS 或 Amazon S3 作为 AWS 存储，然后开始移动数据。

### AWS Storage
[AWS Storage Gateway 的文件接口（又称文件网关）](https://aws.amazon.com/cn/storagegateway/file/)，您可以无缝连接至云，从而将应用程序数据文件和备份映像作为持久对象存储在低成本的 Amazon S3 云存储中。文件网关通过本地缓存实现基于 SMB 或 NFS 访问 Amazon S3 中的数据。

### lsyncd
[lsynd(Live Syncing (Mirror) Daemon)](https://github.com/axkibe/lsyncd)是一个轻量级的实时镜像解决方案，用于将本地目录树同步到远端，本方案中用于将本地基于EBS的文件系统，同步到storage gateway的文件mount（背后是S3的桶）

### 主要步骤：


* 创建datasync task同步源文件系统到s3目标通
* 在开始割接前再次运行datasync task进行增量同步（增加verification选项）
* 创建file gw share或者使用s3fs将目标bucket挂载到各个文件服务器
* 运行lsyncd同步源文件系统和本地mount
* 切换部分流量访问s3进行测试
* 完全切换到s3
* 停止lsyncd同步

## PoC情况：

从PoC结果来看，通过M5.4xlarge的agent可以提供240MB／s的迁移速度，500GB的测试数据，Datasync支持增量同步、filter过滤、文件verification、自定义带宽等功能可以非常方便的满足EBS和S3实时数据同步的要求。

如果从成本方面来考虑，也可以使用开源的s3fs替换file gw。





## 展望：

Datasync在迁移速度，简单易用等方面的表现让人因客户印象深刻。除了在线迁移的场景，后续可以进一步扩展在DR方面的需求，Datasync非常适合做本地文件系统在S3或者EFS上做同步和备份。
