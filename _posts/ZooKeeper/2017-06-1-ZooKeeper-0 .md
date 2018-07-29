---
layout: post
title: Zookeeper视频学习总结
categories: Zookeeper
description: Zookeeper
keywords: Zookeeper
---

# Zookeeper简介 #

Zookeeper是一个高效的分布式协调服务，他暴露一些公用服务，比如命名/配置/同步控制/群组服务等。我们可以使用zk来实现比如达成共识/集群管理/leader选举等。

zookeeper是一个高可用的分布式管理与协调框架，基于ZAB算法（原子消息广播协议）的实现，该框架能够很好的保证分布式环境中数据的一致性。也正是基于这样的特点，使得zk成为了解决分布式一致性问题的利器。

    顺序一致性：从一个客户发起的事务请求，最终会严格地按照其发起的顺序被应用的zk中。

    原子性：所有事务请求的处理结果在整个集群中所有机器上的应用情况是一致的，也就是说要么整个集群所有的机器都成功应用了某个事务，要么有没有应用，一定不会出现部分机器应用了该事务，而另一部分没有应用的情况。

    单一视图：无论客户端连接的是哪个zk服务器，其看到的服务器端数据模型都是一致的。

    可靠性：一旦服务器成功应用了一个事务，并完成对客户端的响应，那么该事务所引起的服务器端状态将会被一致保留下来，除非有另一个事务对其更改

    实时性：通常所说的实时性就是指一旦事务被成功应用，那么客户端就能立刻从服务器获取变更后的数据，zk仅仅能保证在一段时间内，客户端最终一定能从服务器端读取最新的数据状态

# Zookeeper设计目标 #

    目标一：简单的数据结构，zk就是以简单的树形结构来进行互相协调的（也叫树形名字空间）
    
    目标二：可以构建集群。一般zk集群通常由一组机器构成，一般3-5台机器就可以组成一个Zookeeper集群了，只要集群中超过半数以上的机会能够正常工作，那么整个集群就能够正常对外提供服务。
    
    目标三：顺序访问。对于来自每个客户端的每个请求，Zookeeper都会分配一个全局唯一的递增编号，这个标号反应了所有事务操作的先后顺序。应用程序可以使用Zookeeper的这个特性来实现更高层次的同步。
    
    目标四：高性能。由于Zookeeper将全量数据存储在内存中，并直接服务于所有的非事务请求，因此尤其是在读操作为主的场景下性能非常突出。

# Zookeeper的结构 #

Zookeeper会维护一个具有层次关系的数据结构，它非常类似于一个标准的文件系统。

![Zookeeper结构.png]({{ "/images/posts/Zookeeper/Zookeeper-0-1.png" | absolute_url }})


# Zookeeper的数据模型 #

    1. 每个子目录项如NameService都被称为znode，这个znode是被他所在的路径唯一标识，如Server1这个znode的标识为/NameService/Server1
    2. znode可以有子节点目录，并且每个znode可以存储数据，注意EPHEMERAL类型（临时）的目录节点不能有子节点目录
    3. znode是有版本的，每个znode中可以存储的数据可以有多个版本，也就是一个访问路径可以储存多份数据
    4. znode可以是临时节点，一旦创建这个znode的客户端与服务器失去联系，这个znode也将自动删除，Zookeeper的客户端和服务器通信采用长连接方式，每个客户端和服务器通过心跳来保持连接，这个连接状态称为session，如果znode是临时节点，这个session失效，znode也就删除了。
    5. znode的目录名可以自动编号，如App1已经存在，在创建的话，将会自动命名为App2
    6. znode可以被监控，包括这个目录节点中存储的数据的修改，子节点目录的变化等。一旦变化可以通知设置监控的客户端，这个是Zookeeper的核心特性。Zookeeper的很多功能都是基于这个特性实现的，后面在典型应用场景中会有实例介绍。

# Zookeeper组成 #

ZK server根据其身份特性分为三种：leader，follower，observer，其中follower和observer又统称为learner（学习者）

    leader：负责客户端的写请求
    follower：负责客户端的读请求，参与leader选举等
    observer：特殊的"follower"，其可以接受客户端读的请求，但不参与选举。（扩容系统支撑能力，提高了读速度。因为他不接受任何同步地写入请求，只负责与leader同步数据）

# Zookeeper应用场景 #

Zookeeper从设计模式角度来看，是一个基于观察者模式设计的分布式服务管理框架，他负责存储和管理大家都关心的数据，然后接受观察者的注册，一旦这些数据的状态发生变化，Zookeeper就将负责通知已经在Zookeeper上注册的那些观察者做出相应的反应，从而实现集群中类似master/slave管理模式。

## 配置管理 ##

配置的管理在分布式应用环境中很常见，比如我们在平常的应用系统中，经常会碰到这样的需求：如机器的配置列表，运行时开关配置，数据库配置信息，这些全局配置信息通常具备以下3个特性：

    1. 数据量比较小
    2. 数据内容在运行时动态发送变化
    3. 集群中各个节点共享信息，配置一致

## 集群管理 ##

Zookeeper不仅能够帮你维护当前的集群中机器的服务状态，而且能够帮你选出一个leader，让这个leader来管理集群这就是Zookeeper的另一个功能

    1. 希望知道当前集群中究竟有多少机器工作
    2. 对集群中每天集群的运行时状态进行数据收集
    3. 对集群中每台集群进行上下线操作

## 发布与订阅 ##

Zookeeper是一个典型的发布订阅模式的分布式数控管理与协调框架，开发人员可以使用它来进行分布式数据的发布与订阅。

## 数据库切换 ##

比如我们初始化Zookeeper的时候读取其节点上数据库配置文件，当配置一旦发生变更时，Zookeeper就能帮助我们把变更的通知发生到各个客户端，每个客户端在接收到这个变更通知后，就可以重新进行最新消息数据获取。

## 分布式日志的收集 ##

做一个日志收集集群中所有的日志信息，进行统一管理

## 分布式锁，队列管理 ##

Zookeeper的特性就是在分布式场景下高可用。

# Zookeeper集群搭建 #


    1.1 结构：一共三个节点
    
    (zk服务器集群规模不小于3个节点),要求服务器之间系统时间保持一致。
    
    1.2 上传zk 
    
    进行解压： tar zookeeper-3.4.5.tar.gz
    
    重命名： mv zookeeper-3.4.5 zookeeper
    
    修改环境变量： vi /etc/profile
       export ZOOKEEPER_HOME=/usr/local/zookeeper
       export PATH=.:$HADOOP_HOME/bin:$ZOOKEEPER_HOME/bin:$JAVA_HOME/...
    
    刷新： source /etc/profile
    
    到zookeeper下修改配置文件
       cd /usr/local/zookeeper/conf
       mv zoo_sample.cfg zoo.cfg 
    
    修改conf: vi zoo.cfg 修改两处
      （1）dataDir=/usr/local/zookeeper/data
      （2）最后面添加
    	server.0=bhz:2888:3888
    	server.1=hadoop1:2888:3888
    	server.2=hadoop2:2888:3888
    
    服务器标识配置：
       创建文件夹：mkdir data
       创建文件myid并填写内容为0：vi myid (内容为服务器标识 ： 0)
    
    进行复制zookeeper目录到hadoop01和hadoop02还有/etc/profile文件
    
    把hadoop01、hadoop02中的myid文件里的值修改为1和2路径(vi /usr/local/zookeeper/data/myid)
    
    启动zookeeper：路径：/usr/local/zookeeper/bin
       执行：zkServer.sh start(注意这里3台机器都要进行启动)
       状态：zkServer.sh status(在三个节点上检验zk的mode,一个leader和俩个follower)

# Zookeeper操作shell #

    zkCli.sh 进入zookeeper客户端
    根据提示命令进行操作：
    查找：ls / ls /zookeeper
    创建并赋值：create /bhz hadoop
    获取：get /bhz
    设值：set /bhz baihezhuo
    可以看到zookeeper集群的数据一致性
    创建节点有俩种类型：短暂（ephemeral）
    持久（persistent）

# 配置文件zoo.cfg详解 #

    tickTime： 基本事件单元，以毫秒为单位。这个时间是作为 Zookeeper
               服务器之间或客户端与服务器之间维持心跳的时间间隔，
               也就是每隔 tickTime时间就会发送一个心跳。
    
     dataDir：存储内存中数据库快照的位置，顾名思义就是 Zookeeper
              保存数据的目录，默认情况下，Zookeeper
              将写数据的日志文件也保存在这个目录里。
    
     clientPort： 这个端口就是客户端连接 Zookeeper 服务器的端口，
                 Zookeeper会监听这个端口，接受客户端的访问请求。
    
     initLimit： 这个配置项是用来配置 Zookeeper
                 接受客户端初始化连接时最长能忍受多少个心跳时间间隔数，
                 当已经超过 10 个心跳的时间（也就是 tickTime）长度后
                 Zookeeper 服务器还没有收到客户端的返回信息，
                 那么表明这个客户端连接失败。总的时间长度就是
                 10*2000=20 秒。
     
     syncLimit： 这个配置项标识 Leader 与 Follower
			     之间发送消息，请求和应答时间长度，
			     最长不能超过多少个 tickTime
			    的时间长度，总的时间长度就是 5*2000=10 秒
    
     server.A = B:C:D :
	             A表示这个是第几号服务器,
			     B 是这个服务器的 ip 地址；
			     C 表示的是这个服务器与集群中的 Leader
			    服务器交换信息的端口；
			     D 表示的是万一集群中的 Leader
			    服务器挂了，需要一个端口来重新进行选举，选出一个新的 Leader


# java操作zookeeper #

查看示例代码

# Watcher，zk状态，事件类型 #

    zookeeper有watch事件，是一次性触发的，当watch监视的数据发生变化时。通知设置了该watch的client，即watcher.
      同样，其watcher 是监听数据发送了某些变化，那就一定会有对应的事件类型和状态类型。

      事件类型:  (znode节点相关的)
      EventType.NodeCreated
      EventType.NodeDataChanged
      EventType.NodeChildrenChanged
      EventType.NodeDeleted

      状态类型: (是跟客户端实例相关的)
      KeeperState.Disconnected
      KeeperState.SyncConnected
      KeeperState.AuthFailed
      KeeperState.Expired

watcher的特性:一次性、客户端串行执行、轻量。

      一次性:对于ZK的watcher, 你只需要记住一点: zookeeper有watch事件，是一次性触发的，当watch监视的数据发生变化时，通知设置了该watch的client,  即watcher,由于zookeeper 的监控都是一次性的所以每次必须设置监控。

      客户端串行执行:客户端Watcher回调的过程是一个串行同步的过程，这为我们保证了顺序，同时需要开发人员注章一点，千万不要因为一一个Watcher的处理逻辑影响了整个客户端的Watcher回调。

      轻量: WatchedEvent是Zookeeper整 个Watcher通知机制的最小通知单元，整个暑假结构只包含三部分:通知状态、事件类型和节点路径。也就是说Watcher通知非常的简单，只会告诉客户端发生了事件而不会告知其具体内容，需要客户自己去进行获取，比如NodeDataChanged事件， Zookeeper只会通知客户端指定节点的数据发生了变更，  而不会直接提供具体的数据内容。


# zookeeper的ACL(AUTH) #

ACL(Access Control List)Zookeeper作为一个分布式协调框架，其内部存储的都是一些关乎分布式系统运行时状态的元数据，尤其是设计到一些分布式锁、Master选举和协调等应用场景。我们需要有效地保障Zookee per中的数据安全，Zookeeper提供一套完善的ACL权限控制机制来保障数据的安全.

    ZK提供了三种模式。权限模式、投权对象、权限。
    
    权限模式: Scheme,  开发人员最多使用的如下四种权限模式:
    	  IP: ip模式通过ip地址粒度来进行控制权限，例如配置了: ip:192.168.1.107即表示权限控制都是针对这个ip地址的，同时也支持按网段分配。比如1192.168.1.
    	
    	  Digest: digest是最常用的权限控制模式，也更符合我们对权限控制的认识，其类似于"usemame; password形式的权限标设进行权限配置。ZK会对形成的权限标识先后进行俩次编码处理，分别是SHA-1加密算法、BASE64编码。
    	
    	  World: World是一直最开放的权限控制模式。这种模式可以看做为特殊的Digest,他仅仅是一个标识而已。
    	
    	  Super:超级用户模式，在超级用户模式下可以对ZK在意进行操作。
    
    权限对象：指的是权限赋予的用户或者一个指定的实体，例如ip地址或机器等。在不同的模式下，授权对象是不同的，这种模式和权限对象一一对应
    
    权限：权限就是指那些通过权限检测后可以被允许执行的操作，在zk中，对数据的操作权限分为以下五大类：
        CREATE、DELETE、READ、WRITE、ADMIN
 
# zkClient使用 #

查看示例代码

# Curator框架 #

session超时重连，主从选举，分布式计数器，分布式锁等适用于各种复杂的zk场景api封装。

查看示例代码
