---
title: Zookeeper使用记录
tags:
  - java
  - zookeeper
categories:
  - java
date: 2021-07-25 16:59:19
---

### zookeeper命令行使用
``` bash
# 连接zookeeper
$ bin/zkCli.sh -waitforconnection -timeout 3000 -server remoteIP:2181

# 启动服务端
$ bin/zkServer.sh start

# 查看状态
$ bin/zkServer.sh status

# 停止
$ bin/zkServer.sh stop

# 重启
$ bin/zkServer.sh restart
```

- 创建节点 create
``` bash
$ create [-s] [-e] [-c] [-t ttl] path [data] [acl]
# -s 创建有序节点
# -e 创建临时节点

# 创建普通节点
[zk: localhost:2181(CONNECTED) 1] create /test
Created /test
[zk: localhost:2181(CONNECTED) 2] create /test/1 Hello
Created /test/1
[zk: localhost:2181(CONNECTED) 3] get /test/1
Hello

#创建有序节点
[zk: localhost:2181(CONNECTED) 4] create -s /test/seq m1
Created /test/seq0000000001
[zk: localhost:2181(CONNECTED) 5] create -s /test/seq m1
Created /test/seq0000000002
[zk: localhost:2181(CONNECTED) 6] create -s /test/seq m1
Created /test/seq0000000003
[zk: localhost:2181(CONNECTED) 7] ls /test
[1, seq0000000001, seq0000000002, seq0000000003]

#创建临时节点，临时节点会在连接丢失后自动删除
[zk: localhost:2181(CONNECTED) 8] create -e /test/tmp
Created /test/tmp
[zk: localhost:2181(CONNECTED) 9] ls /
[test, zookeeper]
[zk: localhost:2181(CONNECTED) 10] ls /test
[1, seq0000000001, seq0000000002, seq0000000003, tmp]

```

- 列出节点
  
``` bash
$ ls [-s] [-w] [-R] path
# -s to show the stat
# -R to show the child nodes recursely
# -w to set a watch on the child change,Notice: turn on the printwatches
```

- 获取节点信息
  
``` bash
$ get [-s] [-w] path
# -s to show the stat
# -w to set a watch on the data change, Notice: turn on the printwatches
```

- 修改节点
  
``` bash
$ set [-s] [-v version] path data
# -s to show the stat of this node.
# -v to set the data with CAS,the version can be found from dataVersion using stat.
```

- 删除节点
  
``` bash
$ deleteall path [-b batch size]
# 如果有子节点会全部删除
```

- 删除节点单个
  
``` bash
$ delete [-v version] path
```

#### Zookeeper ACL Permissions

- **CREATE**:
  + 可以创建子节点
- **READ**:   
  + 可以读取当前节点
  + 也可以读取其子节点列表
- **WRITE**:   
  + 可以设置当前节点数据
- **DELETE**:   
  + 可以删除子节点
- **ADMIN**:   
  + 可以设置权限

#### Zookeeper ACL Schemes

zookeeper内置Acl方案
- **world** 
  + 只有一个id, anyone, 代表任何人
- **auth** 
  + 使用已添加认证的用户认证
- **digest**
  + 使用用户名密码鉴权
- **ip** 
  + 使用ip地址认证
- **x509** 
  + 使用x509 认证


- 使用digest方案为节点添加权限
  
``` bash
$ echo -n <user>:<password> | openssl dgst -binary -sha1 | openssl base64

# 获取Acl权限
$ getAcl [-s] path

# 设置Acl权限
$ setAcl [-s] [-v version] [-R] path acl

# 添加用户认证
$ addauth scheme auth

# 创建节点并带Acl
$ create [-s] [-e] path data acl

$ echo -n zk:zk | openssl dgst -binary -sha1 | openssl base64
wv1gAVH8RiWIJUyaveCTg5AdOP0=

# 示例
[zk: 192.168.56.101:2181(CONNECTED) 1] setAcl /test/2 digest:zk:wv1gAVH8RiWIJUyaveCTg5AdOP0=:cdrwa
[zk: 192.168.56.101:2181(CONNECTED) 2] ls /test/2
Insufficient permission : /test/2
[zk: 192.168.56.101:2181(CONNECTED) 3] addauth digest zk:zk
[zk: 192.168.56.101:2181(CONNECTED) 4] ls /test/2
[]
```

#### java连接zookeeper

- Curator是Netflix公司开源的一套Zookeeper客户端框架。了解过Zookeeper原生API都会清楚其复杂度。Curator帮助我们在其基础上进行封装、实现一些开发细节，包括接连重连、反复注册Watcher和NodeExistsException等。目前已经作为Apache的顶级项目出现，是最流行的Zookeeper客户端之一。从编码风格上来讲，它提供了基于Fluent的编程风格支持。
- 除此之外，Curator还提供了Zookeeper的各种应用场景：Recipe、共享锁服务、Master选举机制和分布式计数器等

-----

添加pom依赖
``` xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>5.1.0</version>
</dependency>
```

创建curator客户端

``` java
@Component
public class ZookeeperClientConfig {

    private CuratorFramework client;

    @PostConstruct
    public void init() {
        client = createCuratorFramework("127.0.0.1:2181");
        client.start();
    }

    public CuratorFramework createCuratorFramework(String connectString) {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        CuratorFramework curatorFramework = CuratorFrameworkFactory.builder().connectString(connectString)
                .retryPolicy(retryPolicy)
                .connectionTimeoutMs(30000)
                .sessionTimeoutMs(30000)
                .build();
        return curatorFramework;
    }

    @Bean
    public CuratorFramework zookeeperClient() {
        return client;
    }
}
```

-------------------------------

使用curator客户端的基础操作

``` java
@Test
public void testOperation() throws Exception {
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    CuratorFramework client = CuratorFrameworkFactory.builder().connectString("192.168.56.101:2181")
            .retryPolicy(retryPolicy)
            .connectionTimeoutMs(30000)
            .sessionTimeoutMs(30000)
            .namespace("test")
            .build();

    client.start();
    //创建永久节点
    client.create().forPath("/test", "/test data".getBytes());

    //创建永久有序节点
    client.create().withMode(CreateMode.PERSISTENT_SEQUENTIAL).forPath("/test_sequential", "/test_sequential data".getBytes());

    //创建临时节点
    client.create().withMode(CreateMode.EPHEMERAL)
            .forPath("/test/ephemeral", "/test/ephemeral data".getBytes());

    //创建临时有序节点
    client.create().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath("/test/ephemeral_path1", "/test/ephemeral_path1 data".getBytes());

    client.create().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath("/test/ephemeral_path2", "/test/ephemeral_path2 data".getBytes());

    //测试检查某个节点是否存在
    Stat stat1 = client.checkExists().forPath("/test");
    Stat stat2 = client.checkExists().forPath("/test2");

    System.out.println("'/test'是否存在： " + (stat1 != null));
    System.out.println("'/test2'是否存在： " + (stat2 != null));

    //获取某个节点的所有子节点
    System.out.println(client.getChildren().forPath("/"));

    //获取某个节点数据
    System.out.println(new String(client.getData().forPath("/test")));

    //设置某个节点数据
    client.setData().forPath("/test", "/test modified data".getBytes());

    //创建测试节点
    client.create().orSetData().creatingParentContainersIfNeeded()
            .forPath("/test/del_key1", "/test/del_key1 data".getBytes());
    client.create().orSetData().creatingParentContainersIfNeeded()
            .forPath("/test/del_key2", "/test/del_key2 data".getBytes());

    client.create().forPath("/test/del_key2/test_key", "test_key data".getBytes());

    //删除该节点
    client.delete().forPath("/test/del_key1");

    //级联删除子节点
    client.delete().guaranteed().deletingChildrenIfNeeded().forPath("/test/del_key2");
}

```

-------------------------------------

使用PathChildrenCache监听节点变化

``` java
@Test
public void testPathChildrenCache() throws Exception{
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    CuratorFramework client = CuratorFrameworkFactory.builder().connectString("localhost:2181")
            .retryPolicy(retryPolicy)
            .connectionTimeoutMs(30000)
            .sessionTimeoutMs(30000)
            //.namespace("test")
            .build();

    client.start();

    PathChildrenCache cache = new PathChildrenCache(client, "/test", true);

    cache.getListenable().addListener(new PathChildrenCacheListener() {
        @Override
        public void childEvent(CuratorFramework curatorFramework, PathChildrenCacheEvent pathChildrenCacheEvent) throws Exception {
            if(pathChildrenCacheEvent.getType().equals(PathChildrenCacheEvent.Type.INITIALIZED)){
                log.info("子节点初始化成功");
            }else if(pathChildrenCacheEvent.getType().equals(PathChildrenCacheEvent.Type.CHILD_ADDED)){
                log.info("添加子节点路径:"+pathChildrenCacheEvent.getData().getPath());
                if (pathChildrenCacheEvent.getData() != null) {
                    log.info("子节点数据:{}", pathChildrenCacheEvent.getData().getData());
                }
            }else if(pathChildrenCacheEvent.getType().equals(PathChildrenCacheEvent.Type.CHILD_REMOVED)){
                log.info("删除子节点:"+pathChildrenCacheEvent.getData().getPath());
            }else if(pathChildrenCacheEvent.getType().equals(PathChildrenCacheEvent.Type.CHILD_UPDATED)){
                log.info("修改子节点路径:"+pathChildrenCacheEvent.getData().getPath());
                if (pathChildrenCacheEvent.getData() != null) {
                    log.info("修改子节点数据:{}", pathChildrenCacheEvent.getData().getData());
                }
            }
        }
    });
    /*
        * StartMode：初始化方式
        * POST_INITIALIZED_EVENT：异步初始化。初始化后会触发事件
        * NORMAL：异步初始化
        * BUILD_INITIAL_CACHE：同步初始化
        *
        * */
    cache.start(PathChildrenCache.StartMode.POST_INITIALIZED_EVENT);
}
```
运行结果
``` bash
2021-07-25 22:33:35.795  INFO 3360 --- [ChildrenCache-0] s.changer.service.ZooKeeperServiceTest   : 添加子节点路径:/test/test1
2021-07-25 22:33:35.796  INFO 3360 --- [ChildrenCache-0] s.changer.service.ZooKeeperServiceTest   : 子节点数据:null
2021-07-25 22:33:35.796  INFO 3360 --- [ChildrenCache-0] s.changer.service.ZooKeeperServiceTest   : 子节点初始化成功
2021-07-25 22:33:50.142  INFO 3360 --- [ChildrenCache-0] s.changer.service.ZooKeeperServiceTest   : 添加子节点路径:/test/test2
2021-07-25 22:33:50.142  INFO 3360 --- [ChildrenCache-0] s.changer.service.ZooKeeperServiceTest   : 子节点数据:[109, 105, 110, 101]
2021-07-25 22:34:01.019  INFO 3360 --- [ChildrenCache-0] s.changer.service.ZooKeeperServiceTest   : 添加子节点路径:/test/test30000000002
2021-07-25 22:34:01.019  INFO 3360 --- [ChildrenCache-0] s.changer.service.ZooKeeperServiceTest   : 子节点数据:[109, 105, 110, 101]
```

---------------------------------------------

创建分布式可重入锁

``` java
@Test
public void testLock() throws Exception {
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
    CuratorFramework client = CuratorFrameworkFactory.builder().connectString("localhost:2181")
            .retryPolicy(retryPolicy)
            .connectionTimeoutMs(30000)
            .sessionTimeoutMs(30000)
            .namespace("test")
            .build();

    client.start();

    InterProcessMutex lock = new InterProcessMutex(client, "/lock");
    lock.acquire();
    Thread.sleep(10000);
    lock.release();

    client.close();
}
```

观察zookeeper中节点的变化
``` bash
[zk: localhost:2181(CONNECTED) 13] ls /test/lock
[_c_feedfcc0-16a9-4685-9b68-1af077b308a7-lock-0000000002]
[zk: localhost:2181(CONNECTED) 14] ls /test/lock
[_c_feedfcc0-16a9-4685-9b68-1af077b308a7-lock-0000000002]
[zk: localhost:2181(CONNECTED) 15] ls /test/lock
[]
```