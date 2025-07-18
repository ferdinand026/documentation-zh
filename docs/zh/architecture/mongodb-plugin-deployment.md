# TRON MongoDB 事件订阅插件：部署与使用指南

本指南旨在帮助开发者快速部署和使用 TRON MongoDB 事件订阅插件，实现链上事件的实时数据采集、持久化存储与查询服务。文档涵盖系统环境配置、插件部署、数据库安装、查询服务构建及接口调用等全流程。

## 1. 推荐系统配置

为了确保 TRON 节点与事件服务的高效稳定运行，推荐以下配置：

*   **CPU**：16 核及以上
*   **RAM**：32 GB 或更高
*   **SSD**：2.5 TB 以上
*   **操作系统**：Linux 或 macOS


## 2. 系统架构与工作机制
TRON MongoDB 事件订阅系统包含三大核心模块：

1. 事件订阅插件：连接 TRON 节点，捕获事件数据写入 MongoDB。
2. MongoDB 数据库：事件数据持久化存储层。
3. 事件查询服务（TRON Event Query Service）：提供 HTTP API，供外部查询事件数据。


## 3. 部署 TRON 事件订阅插件

### 3.1 插件构建


```shell
git clone https://github.com/tronprotocol/event-plugin.git
cd event-plugin
./gradlew build
```
构建完成后，插件文件位于：
```
event-plugin/build/plugins/plugin-mongodb-*.zip

```


### 3.2 FullNode 节点配置

在 FullNode 配置文件 `config.conf` 中添加如下内容：

```javascript
event.subscribe = {
  version = 1  // 1 表示使用 V2.0 版本，0 表示使用 V1.0 版本。若未配置，默认使用 V1.0。
  startSyncBlockNum = 0 // 历史事件同步的起始区块高度。当 <= 0 时，表示关闭历史事件同步；当 > 0 时，表示开启该功能，并从指定区块高度开始同步历史事件。注：启用该功能时建议使用最新版本的事件插件。

  native = {
    useNativeQueue = false // 是否使用内置消息队列（ZeroMQ）订阅事件。true 表示使用内置消息队列，false 表示使用事件插件。
    bindport = 5555 // 绑定端口
    sendqueuelength = 1000 // 内置消息队列发送队列的最大长度。
  }
  path = "/deploy/fullnode/event-plugin/build/plugins/plugin-mongodb-1.0.0.zip" // 插件的绝对路径，请替换为您的实际路径。
  server = "127.0.0.1:27017" // 目标服务器地址，用于接收事件触发器，通常是 MongoDB 的地址和端口。
  dbconfig = "eventlog|tron|123456" // MongoDB 数据库配置，格式为：数据库名|用户名|密码。
  topics = [
    {
      triggerName = "block" // 触发器名称，不可修改。
      enable = true // 是否启用该事件订阅。
      topic = "block" // MongoDB 中接收事件的集合名称，可修改。
    },
    {
      triggerName = "transaction"
      enable = true
      topic = "transaction"
    },
    {
      triggerName = "contractevent"
      enable = true
      topic = "contractevent"
    },
    {
      triggerName = "contractlog"
      enable = true
      topic = "contractlog"
    }
  ]

  filter = {
    fromblock = "" // 查询范围的起始区块号，可以是空字符串、"earliest" 或指定的区块号。
    toblock = "" // 查询范围的结束区块号，可以是空字符串、"latest" 或指定的区块号。
    contractAddress = [
      "" // 您希望订阅的合约地址。如果设置为空字符串，将接收所有合约地址的日志/事件。
    ]
    contractTopic = [
      "" // 您希望订阅的合约主题。如果设置为空字符串，将接收所有合约主题的日志/事件。
    ]
  }
}
```

**字段解析**：

*   `version`：事件服务框架版本。`1` 表示使用 V2.0 版本，`0` 表示使用 V1.0 版本。若未配置，默认使用 V1.0。
*   `startSyncBlockNum`：V2.0 版本新增支持从本地历史区块中处理并推送事件，可满足用户对历史数据的订阅需求。当 `startSyncBlockNum <= 0` 时，表示关闭历史事件同步功能；当 `startSyncBlockNum > 0` 时，表示开启该功能，并从指定区块高度开始同步历史事件。**注意**：启用该功能时建议使用最新版本的事件插件。
*   `native.useNativeQueue`：是否使用内置消息队列（ZeroMQ）订阅事件。`true` 表示使用内置消息队列，`false` 表示使用插件订阅事件。
*   `native.bindport`：内置消息队列的绑定端口。
*   `native.sendqueuelength`：内置消息队列发送队列的最大长度。
*   `path`：插件的绝对路径，例如 `"/deploy/fullnode/event-plugin/build/plugins/plugin-mongodb-1.0.0.zip"`。
*   `server`：目标服务器地址，用于接收事件触发器，通常是 MongoDB 的地址和端口，例如 `"127.0.0.1:27017"`。
*   `dbconfig`：MongoDB 数据库配置，格式为：`数据库名|用户名|密码`，例如 `"eventlog|tron|123456"`。
*   `topics`：目前支持四种事件类型：`block`、`transaction`、`contractevent` 和 `contractlog`。
    *   `triggerName`：触发器名称，不可修改。
    *   `enable`：是否启用该事件订阅。`true` 为开启，`false` 为禁用。
    *   `topic`：MongoDB 中接收事件的集合名称，可修改。
*   `filter`：事件过滤条件。
    *   `fromblock`：查询范围的起始区块号，可以是空字符串、`"earliest"` 或指定的区块号。
    *   `toblock`：查询范围的结束区块号，可以是空字符串、`"latest"` 或指定的区块号。
    *   `contractAddress`：您希望订阅的合约地址列表。如果设置为空字符串，将接收所有合约地址的日志/事件。
    *   `contractTopic`：您希望订阅的合约主题列表。如果设置为空字符串，将接收所有合约主题的日志/事件。

## 4. MongoDB 安装与配置

MongoDB 将用于存储 TRON 事件数据。请按照以下步骤安装和配置 MongoDB：

### 4.1 安装 MongoDB

首先，创建 MongoDB 的安装目录，并下载、解压 MongoDB 安装包：

```shell
mkdir /home/java-tron
cd /home/java-tron
curl -O https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-4.0.4.tgz
tar zxvf mongodb-linux-x86_64-4.0.4.tgz
mv mongodb-linux-x86_64-4.0.4 mongodb
```

### 4.2 设置环境变量

为了方便后续操作，请设置 MongoDB 的环境变量：

```shell
export MONGOPATH=/home/java-tron/mongodb/
export PATH=$PATH:$MONGOPATH/bin
```

### 4.3 配置 MongoDB

创建 MongoDB 的日志和数据目录，并创建配置文件 `mgdb.conf`：

```shell
mkdir -p /home/java-tron/mongodb/{log,data}
cd /home/java-tron/mongodb/log/ && touch mongodb.log && cd -
vim /home/java-tron/mongodb/mgdb.conf
```

将以下内容写入 `mgdb.conf` 文件中，请确保 `dbpath` 和 `logpath` 使用绝对路径：

```text
dbpath=/home/java-tron/mongodb/data
logpath=/home/java-tron/mongodb/log/mongodb.log
port=27017
logappend=true
fork=true
bind_ip=0.0.0.0
auth=true
wiredTigerCacheSizeGB=2
```

**重要配置说明**：

*   `bind_ip=0.0.0.0`：必须配置为 `0.0.0.0`，否则将拒绝远程连接。
*   `wiredTigerCacheSizeGB`：必须配置此参数以防止内存溢出 (OOM) 问题。

### 4.4 启动 MongoDB

使用配置文件启动 MongoDB 服务：

```shell
mongod --config /home/java-tron/mongodb/mgdb.conf &
```

### 4.5 创建管理员用户和数据库用户

连接到 MongoDB 并创建管理员用户，然后创建用于事件订阅的数据库和用户：

```shell
mongo
use admin
db.createUser({user:"root",pwd:"admin",roles:[{role:"root",db:"admin"}]})

db.auth("root", "admin")
use eventlog
db.createUser({user:"tron",pwd:"123456",roles:[{role:"dbOwner",db:"eventlog"}]})
```

## 5. 部署事件查询服务（ Event Query Service）

TRON Event Query Service 提供 HTTP API 接口，用于查询 MongoDB 中存储的事件数据。该服务依赖 Java 环境。

**注意**：在 Linux Ubuntu 系统上（例如 Ubuntu 16.04.4 LTS），请确保机器的 JDK 使用的是 **Oracle JDK 8**，而不是 OpenJDK 8。您可以参考 [How To Install Java with Apt-Get on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-install-java-with-apt-get-on-ubuntu-16-04) 进行安装。

### 5.1 下载源码

克隆 `tron-eventquery` 项目源码：

```shell
git clone https://github.com/tronprotocol/tron-eventquery.git
cd tron-eventquery
```

### 5.2 构建服务

下载 Maven 并使用 Maven 构建 `tron-eventquery` 服务：

```shell
wget https://mirrors.cnnic.cn/apache/maven/maven-3/3.5.4/binaries/apache-maven-3.5.4-bin.tar.gz --no-check-certificate
tar zxvf apache-maven-3.5.4-bin.tar.gz
export M2_HOME=$HOME/maven/apache-maven-3.5.4
export PATH=$PATH:$M2_HOME/bin
mvn --version
mvn package
```

命令执行成功后，将在 `tron-eventquery/target` 目录下生成 JAR 包，并在 `tron-eventquery/config.conf` 生成配置文件。配置文件内容示例如下：

```text
mongo.host=IP
mongo.port=27017
mongo.dbname=eventlog
mongo.username=tron
mongo.password=123456
mongo.connectionsPerHost=8
mongo.threadsAllowedToBlockForConnectionMultiplier=4
```

请根据您的 MongoDB 配置修改 `mongo.host`、`mongo.port`、`mongo.dbname`、`mongo.username` 和 `mongo.password`。

### 5.3 启动 TRON Event Query Service

启动 `tron-eventquery` 服务并插入索引：

```shell
sh deploy.sh
sh insertIndex.sh
```

**注意**：默认端口为 `8080`。如需修改，请编辑 `deploy.sh` 脚本，例如：

```shell
nohup java -jar -Dserver.port=8081 target/troneventquery-1.0.0-SNAPSHOT.jar 2>&1 &
```

## 6. 启动与验证

完成上述部署步骤后，您可以启动 TRON FullNode 节点并验证事件订阅服务是否正常工作。

### 6.1 启动 FullNode 节点

**重要提示**：在启动 FullNode 节点之前，请确保 MongoDB 服务已成功启动。

启动 FullNode 节点的命令如下：

```shell
java -jar FullNode.jar -c config.conf --es
```

有关 FullNode 节点的安装，请参考 [部署 FullNode](https://tronprotocol.github.io/documentation-zh/using_javatron/installing_javatron/) 文档。

### 6.2 验证插件加载

您可以通过查看 FullNode 日志来验证事件插件是否成功加载：

```shell
tail -f logs/tron.log | grep -i eventplugin
```

如果看到类似以下字样，则表示插件已成功加载：

```text
o.t.c.l.EventPluginLoader 'your plugin path/plugin-kafka-1.0.0.zip' loaded
```

### 6.3 验证数据是否存入 MongoDB

连接到 MongoDB 并查询数据，以验证事件数据是否已从节点获取并通过事件订阅存储到 MongoDB 中：

```shell
mongo 47.90.245.68:27017
use eventlog
db.auth("tron", "123456")
show collections
db.block.find()
```

如果有返回信息，则说明数据已成功存储。否则，请查看 FullNode 日志逐步排查问题。

## 7. 使用 TRON Event Query Service API

TRON Event Query Service 提供了一系列 HTTP API 接口，用于查询 MongoDB 中存储的事件数据。具体API及其用法请参考[Event Query Service HTTP API](https://github.com/tronprotocol/tron-eventquery?tab=readme-ov-file#main-http-service)。
