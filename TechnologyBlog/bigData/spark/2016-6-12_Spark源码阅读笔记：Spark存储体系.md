# 2016-6-12 Spark源码阅读笔记：Spark存储体系

看书读源码个人笔记，以代码阅读过程中的简记为主，点到为止，加快速度，意会即可
接下来尽量用文字总结，少放代码，代码在源码中去看

Spark 版本：1.6.1
-----------------

块管理器BlockManager是Spark存储体系中的核心组件。
Driver Application和Executor都会创建BlockManager。
BlockManager由以下部分组成：
·shuffle客户端ShuffleClient
·BlockManagerMaster，对存在于所有Executor上的BlockManager统一管理
·磁盘块管理器DiskBlockManager
·内存存储MemoryStore
·磁盘存储DiskStore
·外部存储ExternalBlockStore
·非广播Block清理器metadataCleaner和广播Block清理器broadcastCleaner
·压缩算法实现CompressionCodec。
BlockManager要生效，必须要初始化，初始化的步骤：
1）   blockTransferService的初始化和ShuffleClient的初始化。ShuffleClient默认是BlockTransferService，当有外部的ShuffleService时，调用
部的ShuffleService的初始化方法
2）   BlockManagerIdea和ShuffleServiceId的创建，当有外部的ShuffleService时，创建新的BlockManagerId，否则ShuffleServerId默认使用当前Blo
kManager的BlockManagerId
3）   向BlockManagerMaster注册BlockManagerId。