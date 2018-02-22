# 安装VerneMQ

VerneMQ 是一个高性能、分布式的MQTT消息broker。
它支持水平和垂直扩展来支持大量并发的发布者和消费者，同时保持低延迟和容错性。

## Docker 安装 VerneMQ

除了作为可以直接安装到操作系统中的软件包，VerneMQ也可以作为Docker镜像使用。
下面是一个如何使用Docker设置几个VerneMQ节点的例子。
    
### 运行一个*VerneMQ*节点

```docker
docker run --name vernemq1 -d erlio/docker-vernemq
```
配置一下端口转发（MAC为例）：
```docker
docker run -p 1883:1883 --name vernemq1 -d erlio/docker-vernemq
```
这样我们就启动了一个新的VerneMQ节点。
节点默认使用1883端口监听MQTT连接，也可以通过8080端口使用websocket连接MQTT。
不过，此时的broker不能验证连接的客户端，如果想要允许匿名客户端接入，需要配置

"**DOCKER_VERNEMQ_ALLOW_ANONYMOUS=on**"环境变量。

```docker
docker run -e "DOCKER_VERNEMQ_ALLOW_ANONYMOUS=on" --name vernemq1 -d erlio/docker-vernemq
```

### 搭建*VerneMQ*集群

新启动的VerneMQ节点可以通过启动参数自动加入集群。
如果你已经按照上面的例子启动了第一个名为‘vernemq1’的节点，
输入下面的命令将新的节点加入进来。

```docker
docker run -e "DOCKER_VERNEMQ_DISCOVERY_NODE=<IP-OF-VERNEMQ1>" --name vernemq2 -d erlio/docker-vernemq
```
注意：这里输入节点IP的时候不要加尖括号...
你可以使用 *docker inspect <containername/cid> | grep \"IPAddress\"* 命令查询某个docker容器的ip。

### 检查集群状态

可以使用*vmq-admin*命令检查集群状态。

    docker exec vernemq1 vmq-admin cluster show
    +--------------------+-------+
    |        Node        |Running|
    +--------------------+-------+
    |VerneMQ@172.17.0.151| true  |
    |VerneMQ@172.17.0.152| true  |
    +--------------------+-------+

[vmq-admin更多命令](https://vernemq.com/docs/administration/)


## *VerneMQ* 配置

VerneMQ配置文件vernemq.conf根据安装方法和系统的不同所在的路径也不一样。
Linux配置文件默认位置为：*/etc/vernemq/vernemq.conf*。

### 文件格式

* 每行设置一个属性。
* 每行的格式为 `key=value`。
* \#起始的行表示注释，加载时会被忽略。

### 快速入门配置

如果你希望匿名的客户端连接：

* 设置 `allow_anonymous = on`。
* 不要在生产环境打开这个配置。

## Auth using files
### 认证配置 (Authentication)

VerneMQ提供了一个简单的基于文件的密码认证机制，默认启用，
如果你不需要，可以通过设置来禁用它：

    plugins.vmq_passwd = off

生产环境的话，推荐使用基于密码的认证机制，或者可以自己实现认证插件。

    vmq_passwd.password_file = /etc/vernemq/vmq.passwd

检查间隔默认为10秒，也可以在 `vernemq.conf` 中定义。
    
    vmq_passwd.password_reload_interval = 10
    
如果要禁用密码重新加载，可以将 `password_reload_interval` 设置为0

    提示：这两个配置参数也可以在运行时使用vmq-admin命令进行修改。

### 管理*VerneMQ*的密码文件

`vmq-passwd`是一个VerneMQ管理密码文件的工具，
用户名不能包含 “:”，密码的存储格式为crypt(3)。

### 如何使用 *vmq-passwd*

    vmq-passwd [-c | -D] passwordfile username
    
    vmq-passwd -U passwordfile

参数：

`-c`
> 创建一个新的密码文件，如果文件已存在则被覆盖。

`-D`
> 从密码文件中删除指定的用户。

`-U`
> 此选项可用于将使用明文密码的密码文件升级/转换为使用散列(hashed)密码的密码文件。
它不会检测密码是否已被散列，因此在已经包含散列密码的密码文件上使用它将根据
旧的散列生成新的散列，并使密码文件不可用。

`passwordfile`
> 要修改的密码文件

`username`
> 要添加/更新/删除的用户名

### 例子

添加一个用户到一个新的密码文件:(你可以随意命名密码文件，
它只需要和VerneMQ配置文件中的配置相匹配）。

    vmq-passwd -c /etc/vernemq/vmq.passwd henry

从密码文件中删除用户

    vmq-passwd -D /etc/vernemq/vmq.passwd henry

### ACL授权(Authorization)

VerneMQ 提供简单的基于ACL的授权机制，默认开启。如果不需要可以通过配置禁用：

    plugins.vmq_acl = off

VerneMQ 定期检查指定的ACL文件。

    vmq_acl.acl_file = /etc/vernemq/vmq.acl

检查间隔默认为10秒，也可以在vernemq.conf中定义。

    vmq_acl.acl_reload_interval = 10

设置 `acl_reload_interval = 0`来禁用自动重载。

### 管理ACL条目

使用这种格式来添加topic的访问授权：
Topic access is added with lines of the format:

    topic [read|write] <topic>
    
*注意*：在主题和前一个关键词中间只能有一个空格，额外的空格将会被解释为topic的一部分！
另外注意，ACL解释器不支持两个条目之间的空行。

access类型使用`read`和`write`进行控制。
如果为空，则为该topic赋予读和写权限，
topic可以使用MQTT订阅通配符`+`或`#`

第一个topic适用于所有匿名客户端（allow_anonymous = on），
用户特定的ACL被添加到如下所示的配置之后。

    user <username>
    
也可使用pattern关键字替换topic定义ACL：
    
    pattern [read|write] <topic>
    
通配符：

> * `%c` 匹配client id
> * `%u` 匹配用户名

pattern模式是该层级的唯一文本，
即使使用了`user`关键字添加特定用户，
使用pattern定义的ACL也对所有用户生效。

*例子*：

    pattern write sensor/%u/data
    
警告：如果编辑ACL文件并撤销topic的访问授权，VerneMQ不会取消活动的订阅。
（需要重启？验证下）

*简单的ACL例子*：

    # ACL for anonymous clients
    topic bar
    topic write foo
    topic read all


    # ACL for user 'john'
    user john
    topic foo
    topic read baz
    topic write all
  
匿名用户允许：

* 在topic 'bar' 发布、订阅消息
* 在topic 'foo' 发布消息
* 订阅所有topic的消息

用户John允许：

* 在topic 'foo' 发布、订阅消息
* 在topic 'baz' 订阅消息
* 在所有的topic发布消息

### 使用数据库来做验证和授权

### Redis

对于Redis认证和授权，请在vernemq.conf中配置以下内容：

    vmq_diversity.auth_redis.enabled = on
    vmq_diversity.redis.host = 127.0.0.1
    vmq_diversity.redis.port = 6379
    # vmq_diversity.redis.password = 
    # vmq_divserity.redis.database = 0
    
使用`redis-cli`脚本或其他软件库可以添加ACL规则，
`passhash`属性为使用bcrypt加密的客户端密码哈希值，
redis存储key值是encode过的JSON数组，包含挂载点？和用户名还有客户端ID。
请注意，数组项之间不允许存在空格。

    SET "[\"\",\"test-client\",\"test-user\"]" "{\"passhash\":
    \"$2a$12$WDzmynWSMRVzfszQkB2MsOWYQK9qGtfjVpO8iBdimTOjCK/u6CzJK\",
    \"subscribe_acl\":[{\"pattern\":\"a/+/c\"}]}"
    
bcrypt加密版本号支持2a（前缀为‘$2a$’）


## MQTT 选项

这里先介绍下Qos等级

    QoS 0 至多发送一次，发送即丢弃。没有确认消息，也不知道对方是否收到。
    QoS 1 消息至少被传输一次，发布者（客户端/服务器）若因种种异常接收不到PUBACK消息，会再次重新发送PUBLISH消息，同时设置DUP标记为1。接收者以服务器为例，这可能会导致服务器收到重复消息，按照流程，broker（服务器）发布消息到订阅者（会导致订阅者接收到重复消息），然后发送一条PUBACK确认消息到发布者。
          在业务层面，或许可以弥补MQTT协议的不足之处：重试的消息ID一定要一致接收方一定判断当前接收的消息ID是否已经接受过
          但一样不能够完全确保，消息一定到达了。
    Qos 2 确保了仅仅传输接收一次，通信压力稍大些，保证消息到达。

### 重试间隔

设置Qos级别为1和2时消息发送后，VerneMQ未收到响应前重试发送需要等待的时间。

    retry_interval = 20
    
这个选项的默认值为20秒。

### Inflight messages(空中数据)

定义可以同时传输QoS等级1或2消息的最大数量。

    max_inflight_messages = 20
    
默认20条消息，设置为0表示没有限制，这个设置用来做上行保护。

### Load Shedding

设置队列中在线消息的最大数量，默认为1000，设置为-1标识没有限制。
该选项通过丢弃（任意QoS等级）消息来保护客户端会话不超过负荷。

    max_online_messages = 1000
    
设置Qos等级为1和2离线消息的最大数量：
   
    max_offline_messages = 1000
    
同上，-1表示没有限制，0表示一条都不会被存储。

`max_inflight_messages`和`max_online/offline_messages`的区别在于，
前着保护上行，后者保护下行。

## MQTT 监听（Listeners）

监听器指定VerneMQ绑定哪个ip地址和端口，根据选择的传输协议不同必须提供不同的参数。
VerneMQ允许以分层方式编写监听器配置，非常灵活。初始的配置文件有一些默认配置，
请按自己的需要进行重写。

    # defines the default nr of allowed concurrent 
    # connections per listener
    listener.max_connections = 10000
    
    # defines the nr. of acceptor processes waiting
    # to concurrently accept new connections
    listener.nr_of_acceptors = 10
    
    # used when clients of a particular listener should
    # be isolated from clients connected to another 
    # listener.
    listener.mountpoint = off

以上是适用于所有传输协议的默认配置，也是仅有的几个与TCP和WebSocket监听器有关的配置。

这些全局配置可以专用在某种传输协议或某个listener上：
通过override`listener.tcp.CONFIG = VAL`和`listener.tcp.LISTENER.CONFIG = VAL`
`LISTENER`表示要进一步配置哪个listener。

### 配置例子

TCP监听8883端口，WebSocket监听8888端口：

    listener.tcp.default = 127.0.0.1:1883
    listener.ws.default = 127.0.0.1:8888

可以使用不同的`name`添加新的监听器。在上面的配置中，`name`为`default`。
下面示范如何定义一个新的listener，已经限制这个listener的最大连接数。

    listener.tcp.my_other = 127.0.0.1:18884
    listener.tcp.my_other.max_connections = 100

### PROXY协议

VerneMQ的listener可以配置为接受支持proxy协议的代理服务器的连接。
这使VerneMQ能够检索源IP /端口等对等信息，而且如果使用代理来做TLS终结，
则还可以检索TLS客户端证书等详细信息。

通过配置 `listener.tcp.proxy_protocol = on` 或 `listener.tcp.LISTENER.proxy_protocol = on.`
来为TCP listener启用PROXY协议。

### 配置例子

在8883端口接受SSL连接：

    listener.ssl.cafile = /etc/ssl/cacerts.pem
    listener.ssl.certfile = /etc/ssl/cert.pem
    listener.ssl.keyfile = /etc/ssl/key.pem

    listener.ssl.default = 127.0.0.1:8883

打开客户端认证：

    listener.ssl.require_certificate = on

如果您使用客户端证书并且想要使用证书CN值作为用户名，您可以设置：

    listener.ssl.use_identity_as_username = on

`require_certificate` 和 `use_identity_as_username` 这两个选项的默认值都是`off`。

同样的配置可以用来加密WebSocket连接，只需要使用`wss`作为协议标识。

    listener.wss.require_certificate = on

提示：使用SSL时，仍然需要配置身份验证和授权。也就是说，将'allow_anonymous'设置为'off'，然后配置vmq_acl和vmq_passwd或身份验证插件。

## HTTP监听器

跳过

## 非标准MQTT选项

### ClientId 最大长度

设置client id的最大长度，MQTT v3.1指定了23个字符的限制。

    max_client_id_size = 23

此选项默认值为23。

### 过期持久的无效连接

通过设置可以将那些被`clean_session`设置为false的持久客户端（persistent client）如果在特定时间内不reconnect全部删除。

警告：这是一个非标准选项。就MQTT协议而言，持久的客户端永远是持久的。

有效期需要被设置为整数(integer)
1h（一小时），1d（一天），1w（一周），1m（1个月），1y（1年），never（用不）

    persistent_client_expiration = 1w

此选项默认值为`never`。

### 消息体大小限制

限制VerneMQ允许的最大发布有效payload大小（以字节为单位）。 超过这个大小的信息将被拒绝。

    message_size_limit = 0

此选项默认值为0，表示不限制消息大小。MQTT协议标准的payload大小为268435455字节

## WebSocket

VerneMQ同样支持WebSocket协议。想要通过WebSocket连接VerneMQ，需要在vernemq.conf中配置：

    #普通ws
    listener.ws.default = 127.0.0.1:8888
    #或通过ssl
    listener.wss.default = 127.0.0.1:8889

建立连接时请在ws的url后面添加`/mqtt`路径。

## 日志

### 控制台日志（console logging）

VerneMQ打印console log的位置：

    log.console = off | file | console | both

VerneMQ默认将日志输出到文件中：

    log.console.file = /path/to/log/file

log文件的默认名称为`console.log`。

console log日志级别：

    log.console.level = debug | info | warning | error

### 错误日志（error logging）

VerneMQ默认将错误日志输出到文件：

    log.error.file = /path/to/log/file

错误日志文件的默认名称为`error.log`。

### 系统日志（SysLog）

系统日志开关：

    log.syslog = on

### 消费者会话负载均衡（Consumer Session Balancing）

有时消费者会被他们收到的信息数量所淹没。
VerneMQ可以对订阅同一主题的、使用同一clientId的消费者实例进行负载均衡。

### 开启会话负载均衡（Session Balancing）

在vernemq.conf中配置：

    allow_multiple_sessions = on 
    queue_deliver_mode = balance

注意：这个设置将在每个节点上激活全局Session Balancing。
如果希望针对特定的消费者进行Balancing需要安装一个插件。
如果消费者分布在不同集群节点上，此负载均衡不会生效。

### 管理插件

### Enable a plugin
 -- 略

### Disable a plugin
 -- 略

### 共享订阅（share subscriptions）

共享订阅是一种将消息分发给一组共享订阅者的机制。
这种机制与普通订阅机制的不同是:
* 共享订阅机制保证每条消息只被一个订阅者接收到。
* 普通订阅机制每个订阅者都能收到发布消息的副本。

共享订阅的topic格式为`$share/sharename/topic`,消息将根据定义的分配策略进行分配。

Tip: 当使用命令行订阅共享topic时，有些命令行shell会将topic的$share关键字扩展为环境变量，请注意。

### 共享订阅配置

共享订阅目前支持三种分发策略：
* perfer_local：
    消息优先随机发布给本地客户端，如果不存在则发布给远程客户端。
* random：
    消息随机发布给一个客户端。
* local_only：
    消息随机发布给本地客户端。

    `shared_subscription_policy = prefer_local`

## 高级设置

### 隐藏设置

vernemq中有一些隐藏设置，这些设置需要手动添加到vernemq.conf中。

### Queue Deliver mode

设置当有多个会话时队列如何传递消息。

* fanout：所有连接的会话都会收到消息。
* balance：随机选择一个连接的会话。

    queue_deliver_mode = balance

### Queue Type

设置队列如何处理消息。
* fifo：默认设置，先进先出。
* lifo：后进先出。

    queue_type = fifo

### 最大消息速率（Max Message Rate）

设置每个会话每秒的最大发布速率。默认值为0表示不限制速率。如果设置为2意味着每个发布者每秒只能发布两条消息。

    max_message_rate = 2

### 设置最大迁移时间（Max Drain Time）

由于订阅者存储的最终一致性，有可能在队列迁移期间仍旧在旧集群节点上驱动消息。 
使用此参数，可以通过在将队列迁移到其他集群节点之后保持队列一段时间（以秒为单位）。

    max_drain_time = 20

### 设置每个迁移步骤的最大消息数（Max Msgs per Drain Step）

设置每个迁移step允许传递到远程节点的最大消息数。
较大的值将会加快队列的迁移，但会增加带宽的消耗。

    max_msgs_per_drain_step = 1000

### Default Reg View

这块不太理解。
允许选择一个新的默认reg_view。reg_view是路由消息的预定义方式。 
可以加载和使用多个view，但必须选择一个作为默认view。 
默认路由是`vmq_reg_trie`，即通过内置的trie数据结构进行路由。

    vmq_reg_view = "vmq_reg_trie"

### Reg Views

启动过程中加载的view列表。 它只用于在路由reg_views之间动态选择的插件。

    reg_views = "[vmq_reg_trie]"

### Outgoing Clustering Buffer Size

指定在远程节点不可用的情况下缓冲的字节大小，默认值为10000。整数。

    outgoing_clustering_buffer_size = 15000

## LevelDB

VerneMQ 使用的是Google的LevelDB作为存储消息和订阅者信息的快速存储。每个VerneMQ节点都运行着独立的内嵌的LevelDB。

### Configuration of LevelDB memory

唯一你需要了解的是LevelDB管理着它自己的内存。这意味着VerneMQ不需要为LevelDB分配或释放内存。
你只需要在vernemq.conf中配置一个值来告诉LevelDB可以使用多少比例的内存。

    leveldb.maximum_memory.percent = 20

## MQTT 桥接（MQTT Bridge）

桥接是一种非标准的方式，尽管将单个broker连接到另一个broker是MQTT中的一种实现标准。
这种方式允许远程broker的topic树成为本地broker的一部分。
VerneMQ支持p托尼盖的TCP连接和SSL连接。

### 启用桥接功能

在VerneMQ中，桥接功能是插件的形式，默认是关闭的。配置桥接后，请确认开启了桥接插件。

    plugins.vmq_bridge = on

关于插件的使用，请移步到插件管理（Managing plugins）一节。

### 例子

桥接远程的broker：

    vmq_bridge.tcp.br0 = 192.168.1.12:1883

其他的连接参数：

    # use a clean session (defaults to 'off')
    vmq_bridge.tcp.br0.cleansession = off | on

    # set the client id (defaults to 'auto', which generates one)
    vmq_bridge.tcp.br0.client_id = auto | my_bridge_client_id

    # set keepalive interval (defaults to 60 seconds)
    vmq_bridge.tcp.br0.keepalive_interval = 60

    # set the username and password for the bridge connection
    vmq_bridge.tcp.br0.username = my_bridge_user
    vmq_bridge.tcp.br0.password = my_bridge_pwd

    # set the restart timeout (defaults to 10 seconds)
    vmq_bridge.tcp.br0.restart_timeout = 10

    # VerneMQ indicates other brokers that the connection
    # is established by a bridge instead of a normal client.
    # This can be turned off if needed:
    vmq_bridge.tcp.br0.try_private = off

