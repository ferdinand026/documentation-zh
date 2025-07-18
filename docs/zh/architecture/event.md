# 事件订阅
TRON 区块链提供强大的事件订阅机制，使开发者能够实时或历史性地捕获链上的关键事件信息，如交易状态、合约调用、区块生成等。根据应用场景的不同，TRON 提供了两种主要的事件订阅方式：通过**本地事件插件**和通过 `java-tron` **内置消息队列（ZeroMQ）**。

## 事件订阅方式概览

### 1. 本地事件插件订阅（推荐）
该方式基于可扩展的插件架构，支持将链上事件实时或批量写入外部系统（如数据库、Kafka 消息队列），适合对事件持久化、分析或高可靠性要求较高的场景。

**优势：**

* 支持多种插件类型：目前支持 Kafka 和 MongoDB 插件。
* 可订阅多种数据类型：包括区块、交易、智能合约事件以及合约日志等。
* 支持复杂的事件过滤条件
* 可配合 [tron-eventquery](https://github.com/tronprotocol/tron-eventquery) 提供在线查询服务
* 支持历史事件回溯（V2.0）

> ⚠️ 推荐用于生产环境，尤其是在对数据完整性和分析有要求的场景下。

### 2. 内置消息队列订阅（ZeroMQ）
Java-tron 节点内置 ZeroMQ 消息队列，提供轻量级事件推送能力，开发者无需部署额外插件即可使用。适用于对实时性要求较高但不需要存储或回溯事件的简单场景。

**特点：**

* 无需部署插件，直接连接 TRON 节点订阅事件
* 支持实时推送，但不支持事件存储或回溯
* 适合快速开发和测试环境使用


## 事件服务框架（基于本地插件）
基于本地插件的事件订阅方式依赖于事件服务框架，java-tron 当前支持两个版本：V1.0 和 V2.0。

**✳️ V2.0 框架优势**
与 V1.0 相比，V2.0 引入了 历史事件回溯推送能力，支持用户订阅任意历史区块事件。V1.0 仅支持新区块产生时的实时事件推送。

🔗 查看详细对比与选择建议：[事件服务框架 V2.0 介绍](https://medium.com/tronnetwork/event-service-framework-v2-0-0622f2f07249)

### 🛠 框架工作流程如下：
1. 节点从区块链中获取事件信息；
2. 将事件封装并写入事件缓存队列；
3. 插件模块异步消费队列数据；
4. 插件将数据推送到指定目标系统（如 Kafka、MongoDB）；
5. 上层业务应用可持续消费这些事件数据

## 事件类型

TRON 事件服务支持多种类型的事件订阅。开发者可以通过修改配置文件，灵活订阅所需的事件类型。目前支持以下四种主要事件类型：

1.交易事件
订阅内容包括：
```
transactionId: 交易哈希
blockHash: 区块哈希
blockNumber: 区块高度
energyUsage: 此次调用中，合约调用者消耗的Energy的总量
energyFee: 此次调用中，合约调用者消耗的Energy中，需要TRX支付的数目(SUN为单位)
originEnergyUsage: 此次调用中，合约开发者消耗的Energy的总量
energyUsageTotal: 此次调用中，合约调用者和合约开发者消耗的Energy的总量
```

2.区块事件
订阅内容包括：
```
blockHash: 区块哈希
blockNumber: 区块高度
transactionSize: 区块中包含的交易的数目
latestSolidifiedBlockNumber: 最新的固化块的高度
transactionList: 交易哈希列表
```

3.合约事件
订阅内容包括：
```
transactionId: 交易哈希
contractAddress: 合约地址
callerAddress: 合约调用者地址
blockNumber: 合约事件所在的区块高度
blockTimestamp: 区块时间戳
eventSignature: 事件签名
topicMap: the map of topic in solidity language
data: the data information in solidity language
```

4.合约日志事件
订阅内容包括：
```
transactionId: 交易哈希
contractAddress: 合约地址
callerAddress: 合约调用者地址
blockNumber: 合约事件所在的区块高度
blockTimestamp: 区块时间戳
contractTopics: the list of topic in solidity language
data: the data information in solidity language
removed: 'true'代表日志已经被移除
```

**合约事件**与**合约日志事件**订阅支持以下过滤功能：

```
fromBlock: 起始区块索引
toBlock: 结束区块索引
contractAddress: 合约地址
contractTopics: contract topics列表
```

**重要提示**：

*   在订阅非固化事件时，请务必以 `blockNumber` 和 `blockHash` 两个参数为准进行验证。在网络连接不稳定等特殊情况下，可能发生链分叉导致事件重组，从而使部分事件失效。



## 如何迁移至新版事件服务（V2.0）
新版事件服务框架（V2.0）引入了对历史事件回溯的支持，并优化了事件推送机制。若你希望利用更高效、更稳定的事件处理能力，此指南将帮助你平滑完成迁移。
### 迁移前的关键评估
在进行迁移之前，请根据以下因素评估当前业务是否适合升级至 V2.0：
1. 是否依赖内部交易日志？
   当前 V2.0 暂不支持内部交易日志，即事件中的 internalTransactionList 字段为空。如果你的业务依赖该字段（如内部调用追踪），建议继续使用 V1.0，等待后续版本支持该功能。

2. 是否使用了较旧版本的事件插件？
   建议同步升级事件插件至最新版。旧版本插件在处理大量历史事件时可能出现性能瓶颈或内存问题，从而影响数据同步与消费效率。

### 迁移操作步骤

#### 步骤1：获取并编译新版事件插件
你可以从 GitHub 获取源码并自行构建，或直接下载官方 release 版本：
**方式一：源码构建**
```
git clone git@github.com:tronprotocol/event-plugin.git
cd event-plugin
git checkout master

./gradlew build  
```
构建完成后，生成的 .zip 文件即为插件包。

**方式二：下载官方发布版本**
访问 [event-plugin Releases 页面](https://github.com/tronprotocol/event-plugin/releases) 下载最新版插件包。



#### 步骤2：修改 FullNode 配置启用新版事件服务

在 config.conf 文件中添加或修改以下配置项以启用新版事件服务：

```
event.subscribe.version = 1    # 1 means v2.0 , 0 means v1.0
```

#### 步骤 3：配置事件插件
新版事件服务的插件配置方式与旧版基本一致，你可以参考以下文档完成插件部署：
* [事件插件部署(MongoDB)](https://cn.developers.tron.network/docs/event-plugin-deployment-mongodb)
* [事件插件部署(Kafka)](https://cn.developers.tron.network/docs/event-plugin-deployment-kafka)

#### 步骤 4（可选）：配置历史事件同步起点

如果你希望从指定区块高度开始同步历史事件，请在配置文件中加入：

```
event.subscribe.startSyncBlockNum = <block_height>
```

#### 步骤 5：启动 FullNode 和插件
完成上述配置后，使用以下命令启动 FullNode 并加载事件插件：

```
java -jar FullNode.jar -c config.conf --es
```

