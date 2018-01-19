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
你也许需要配置一下端口转发（以MAC为例）：
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

### 组建*VerneMQ*集群

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

//todo