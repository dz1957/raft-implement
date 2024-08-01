# Raft-Impl源码阅读

> Raft-Impl是一个共识算法Raft的实现项目，使用Java语言编写。详见[项目Github地址](https://github.com/wenweihu86/raft-java)。

> 本文档是Raft-Impl项目的源码阅读文档。



## 客户端

### 执行逻辑示例

#### ConcurrentClientMain类

##### 主方法main()

```java
String ipPorts = args[0];
RpcClient rpcClient = new RpcClient(ipPorts);
ExampleService exampleService = BrpcProxy.getProxy(rpcClient, ExampleService.class);
```

> 从启动命令中提取服务端的IP地址和端口号参数，并创建相应的RPC客户端，获取代理对象。



```java
ExecutorService readThreadPool = Executors.newFixedThreadPool(3);
ExecutorService writeThreadPool = Executors.newFixedThreadPool(3);
Future<?>[] future = new Future[3];
for (int i = 0; i < 3; i++) {
    future[i] = writeThreadPool.submit(new SetTask(exampleService, readThreadPool));
}
```

> 创建两个线程池，分别处理读写任务。



##### 写任务内部类**SetTask**的run()方法

```java
ExampleProto.SetRequest setRequest = ExampleProto.SetRequest.newBuilder()
        .setKey(key).setValue(value).build();
ExampleProto.SetResponse setResponse = exampleService.set(setRequest);
```

> 创建写请求，交由RPC客户端处理。



```java
if (setResponse != null) {
    ...
    readThreadPool.submit(new GetTask(exampleService, key));
} else {
    ...
}
```

> 如果写请求得到响应，将读任务提交到线程池，读取写请求的结果。



##### 读任务内部类**GetTask**的run()方法

```java
ExampleProto.GetRequest getRequest = ExampleProto.GetRequest.newBuilder()
        .setKey(key).build();
ExampleProto.GetResponse getResponse = exampleService.get(getRequest);
```

> 创建读请求，交由RPC客户端处理。





## 服务端

### 执行逻辑示例

#### ServerMain类

##### 主方法main()

```java
String dataPath = args[0];
String servers = args[1];
String[] splitArray = servers.split(",");
List<RaftProto.Server> serverList = new ArrayList<>();
for (String serverString : splitArray) {
    RaftProto.Server server = parseServer(serverString);
    serverList.add(server);
}
RaftProto.Server localServer = parseServer(args[2]);
```

> 从启动命令中提取参数，包括本地数据存储路径、集群节点信息、当前节点信息。



```java
RpcServer server = new RpcServer(localServer.getEndpoint().getPort());
```

> 创建RPC服务端。



```java
RaftOptions raftOptions = new RaftOptions();
raftOptions.setDataDir(dataPath);
raftOptions.setSnapshotMinLogSize(10 * 1024);
raftOptions.setSnapshotPeriodSeconds(30);
raftOptions.setMaxSegmentFileSize(1024 * 1024);
```

> 配置Raft相关参数。



```java
ExampleStateMachine stateMachine = new ExampleStateMachine(raftOptions.getDataDir());
```

> 创建复制状态机。



```java
RaftNode raftNode = new RaftNode(raftOptions, serverList, localServer, stateMachine);
```

> 通过已经获取的Raft相关参数、集群节点信息集合、当前节点信息、复制状态机，创建Raft节点。



```java
RaftConsensusService raftConsensusService = new RaftConsensusServiceImpl(raftNode);
server.registerService(raftConsensusService);
```

> 创建Raft节点相互通信的服务并注册。



```java
RaftClientService raftClientService = new RaftClientServiceImpl(raftNode);
server.registerService(raftClientService);
```

> 创建客户端访问当前Raft节点的服务并注册。



```java
ExampleService exampleService = new ExampleServiceImpl(raftNode, stateMachine);
server.registerService(exampleService);
```

> 创建当前Raft节点的应用服务并注册。



```java
server.start();
raftNode.init();
```

> 启动RPC服务端，并初始化Raft节点。



##### 解析节点信息的parseServer()方法

```java
String[] splitServer = serverString.split(":");
String host = splitServer[0];
Integer port = Integer.parseInt(splitServer[1]);
Integer serverId = Integer.parseInt(splitServer[2]);
```

> 解析启动参数中的节点信息字符串，格式为"IP地址:端口号:节点ID"。



```java
RaftProto.Endpoint endPoint = RaftProto.Endpoint.newBuilder()
        .setHost(host).setPort(port).build();
RaftProto.Server.Builder serverBuilder = RaftProto.Server.newBuilder();
RaftProto.Server server = serverBuilder.setServerId(serverId).setEndpoint(endPoint).build();
return server;
```

> 根据IP地址和端口号创建终端，根据终端和节点ID创建服务端实例。



### 组件解析

#### 复制状态机ExampleStateMachine类

##### 继承自StateMachine接口

```java
public interface StateMachine {
    void writeSnapshot(String snapshotDir);
    
    void readSnapshot(String snapshotDir);
    
    void apply(byte[] dataBytes);
}
```

> 定义了三个核心接口方法，分别为：获取复制状态机的快照并写入到磁盘文件，从磁盘文件读取快照并加载到内存，将数据应用到复制状态机。



##### 成员属性及初始化

```java
static {
    RocksDB.loadLibrary();
}

private RocksDB db;
private String raftDataDir;

public ExampleStateMachine(String raftDataDir) {
    this.raftDataDir = raftDataDir;
}
```

> 初始化数据文件的磁盘存储路径，并且使用了RocksDB作为数据存储的实现。



##### writeSnapshot()方法

```java
Checkpoint checkpoint = Checkpoint.create(db);
checkpoint.createCheckpoint(snapshotDir);
```

> 创建快照并写入磁盘。



##### readSnapshot()方法

```java
String dataDir = raftDataDir + File.separator + "rocksdb_data";
File dataFile = new File(dataDir);
if (dataFile.exists()) {
    FileUtils.deleteDirectory(dataFile);
}
File snapshotFile = new File(snapshotDir);
if (snapshotFile.exists()) {
    FileUtils.copyDirectory(snapshotFile, dataFile);
}
```

> 使用目标快照文件替换数据文件。



```java
db = RocksDB.open(options, dataDir);
```

> 打开数据库，初始化数据存储的实现。



##### apply()方法

```java
ExampleProto.SetRequest request = ExampleProto.SetRequest.parseFrom(dataBytes);
db.put(request.getKey().getBytes(), request.getValue().getBytes());
```

> 将字节数据解析为写请求，写入数据库。



##### 根据读请求构造读响应的get()方法

```java
ExampleProto.GetResponse.Builder responseBuilder = ExampleProto.GetResponse.newBuilder();
byte[] keyBytes = request.getKey().getBytes();
byte[] valueBytes = db.get(keyBytes);
if (valueBytes != null) {
    String value = new String(valueBytes);
    responseBuilder.setValue(value);
}
ExampleProto.GetResponse response = responseBuilder.build();
return response;
```

> 根据请求中的Key，从数据库中读取Value，封装为响应。



#### 节点的应用服务ExampleServiceImpl类

##### 继承自ExampleService接口

```java
public interface ExampleService {
    ExampleProto.SetResponse set(ExampleProto.SetRequest request);

    ExampleProto.GetResponse get(ExampleProto.GetRequest request);
}
```

> 定义了两个核心接口方法，分别为：根据写请求构造写响应，根据读请求构造读相应。



##### 成员属性及初始化

```java
private RaftNode raftNode;
private ExampleStateMachine stateMachine;
private int leaderId = -1;
private RpcClient leaderRpcClient = null;
private Lock leaderLock = new ReentrantLock();

public ExampleServiceImpl(RaftNode raftNode, ExampleStateMachine stateMachine) {
    this.raftNode = raftNode;
    this.stateMachine = stateMachine;
}
```

> 维护了Leader的节点ID和与Leader进行通信的RPC客户端，以及用于更新Leader信息的同步锁。
>
> 初始化Raft节点和复制状态机。



##### 更新Leader信息的onLeaderChangeEvent()方法

```java
if (raftNode.getLeaderId() != -1
        && raftNode.getLeaderId() != raftNode.getLocalServer().getServerId()
        && leaderId != raftNode.getLeaderId()) {
    leaderLock.lock();
    ...
    leaderLock.unlock();
}
```

> 同时满足三个条件时才需要更新Leader信息：真实的Leader的节点ID不为-1，当前节点不是Leader节点，以及当前存储的Leader的节点ID不是真实的Leader的节点ID。
>
> 更新Leader信息的逻辑使用同步锁保护。



```java
if (...) {
    leaderLock.lock();
    if (leaderId != -1 && leaderRpcClient != null) {
        leaderRpcClient.stop();
        leaderRpcClient = null;
        leaderId = -1;
    }
	...
    leaderLock.unlock();
}
```

> 如果当前存储的Leader的节点ID不为-1，且与Leader进行通信的RPC客户端不为空，即维护了过期Leader信息，则关闭RPC客户端并清空过期信息。



```java
if (...) {
    leaderLock.lock();
	...
    leaderId = raftNode.getLeaderId();
	Peer peer = raftNode.getPeerMap().get(leaderId);
	Endpoint endpoint = new Endpoint(peer.getServer().getEndpoint().getHost(),
        	peer.getServer().getEndpoint().getPort());
	leaderRpcClient = new RpcClient(endpoint, rpcClientOptions);
    leaderLock.unlock();
}
```

> 获取真实的Leader的节点ID，进而获取Leader的IP地址和端口号，创建与Leader进行通信的RPC客户端。



##### set()方法

```java
ExampleProto.SetResponse.Builder responseBuilder = ExampleProto.SetResponse.newBuilder();
if (raftNode.getLeaderId() <= 0) {
    ...
} else if (raftNode.getLeaderId() != raftNode.getLocalServer().getServerId()) {
    ...
} else {
    ...
}
ExampleProto.SetResponse response = responseBuilder.build();
return response;
```

> 根据维护的Leader的节点ID，对写请求进行相应处理并构造响应。



```java
if (raftNode.getLeaderId() <= 0) {
    responseBuilder.setSuccess(false);
} else if (raftNode.getLeaderId() != raftNode.getLocalServer().getServerId()) {
    ...
} else {
    ...
}
```

> 如果维护的Leader的节点ID为负，表明此节点目前没有Leader的信息，无法处理写请求，直接进行失败响应。



```java
if (raftNode.getLeaderId() <= 0) {
    ...
} else if (raftNode.getLeaderId() != raftNode.getLocalServer().getServerId()) {
    onLeaderChangeEvent();
    ExampleService exampleService = BrpcProxy.getProxy(leaderRpcClient, ExampleService.class);
    ExampleProto.SetResponse responseFromLeader = exampleService.set(request);
    responseBuilder.mergeFrom(responseFromLeader);
} else {
    ...
}
```

> 如果当前节点不是Leader节点，先检查是否需要更新Leader信息，根据RPC客户端获取代理对象，向Leader发送RPC，并使用Leader的响应结果构造响应。



```java
if (raftNode.getLeaderId() <= 0) {
    ...
} else if (raftNode.getLeaderId() != raftNode.getLocalServer().getServerId()) {
    ...
} else {
    byte[] data = request.toByteArray();
    boolean success = raftNode.replicate(data, RaftProto.EntryType.ENTRY_TYPE_DATA);
    responseBuilder.setSuccess(success);
}
```

> 如果当前节点是Leader节点，则需要调用**RaftNode**类的**replicate()**方法，将写请求进行集群写入，并构造响应。



##### get()方法

```java
ExampleProto.GetResponse response = stateMachine.get(request);
return response;
```

> 调用**ExampleStateMachine**类的**get()**方法，根据读请求构造读响应。



#### 节点提供客户端访问的服务RaftClientServiceImpl类

##### 继承自RaftClientService接口

```java
public interface RaftClientService {
    RaftProto.GetLeaderResponse getLeader(RaftProto.GetLeaderRequest request);

    RaftProto.GetConfigurationResponse getConfiguration(RaftProto.GetConfigurationRequest request);

    RaftProto.AddPeersResponse addPeers(RaftProto.AddPeersRequest request);

    RaftProto.RemovePeersResponse removePeers(RaftProto.RemovePeersRequest request);
}
```

> 定义了四个核心接口方法，分别为：获取集群Leader信息，获取集群配置信息，新增节点，移除节点。



##### 成员属性及初始化

```java
private RaftNode raftNode;

public RaftClientServiceImpl(RaftNode raftNode) {
    this.raftNode = raftNode;
}
```

> 初始化Raft节点。



##### getLeader()方法

```java
RaftProto.GetLeaderResponse.Builder responseBuilder = RaftProto.GetLeaderResponse.newBuilder();
responseBuilder.setResCode(RaftProto.ResCode.RES_CODE_SUCCESS);
RaftProto.Endpoint.Builder endPointBuilder = RaftProto.Endpoint.newBuilder();

raftNode.getLock().lock();
try {
    ...
} finally {
    raftNode.getLock().unlock();
}

responseBuilder.setLeader(endPointBuilder.build());
RaftProto.GetLeaderResponse response = responseBuilder.build();
return responseBuilder.build();
```

> 对获取Leader信息的请求进行相应处理并构造响应。
>
> 处理逻辑使用同步锁保护。



```java
try {
    int leaderId = raftNode.getLeaderId();
    if (leaderId == 0) {
        responseBuilder.setResCode(RaftProto.ResCode.RES_CODE_FAIL);
    } else if (leaderId == raftNode.getLocalServer().getServerId()) {
        ...
    } else {
        ...
    }
} finally {
    ...
}
```

> 如果获取的Leader的节点ID为0，则表示获取失败。



```java
try {
    int leaderId = raftNode.getLeaderId();
    if (leaderId == 0) {
        ...
    } else if (leaderId == raftNode.getLocalServer().getServerId()) {
        endPointBuilder.setHost(raftNode.getLocalServer().getEndpoint().getHost());
        endPointBuilder.setPort(raftNode.getLocalServer().getEndpoint().getPort());
    } else {
        ...
    }
} finally {
    ...
}
```

> 如果当前节点是Leader节点，则响应当前节点的IP地址和端口号。



```java
try {
    int leaderId = raftNode.getLeaderId();
    if (leaderId == 0) {
        ...
    } else if (leaderId == raftNode.getLocalServer().getServerId()) {
        ...
    } else {
        RaftProto.Configuration configuration = raftNode.getConfiguration();
        for (RaftProto.Server server : configuration.getServersList()) {
            if (server.getServerId() == leaderId) {
                endPointBuilder.setHost(server.getEndpoint().getHost());
                endPointBuilder.setPort(server.getEndpoint().getPort());
                break;
            }
        }
    }
} finally {
    ...
}
```

> 如果获取到Leader的节点ID，但当前节点不是Leader节点，则获取节点列表并遍历，找到Leader节点，响应Leader节点的IP地址和端口号。



##### getConfiguration()方法

```java
RaftProto.GetConfigurationResponse.Builder responseBuilder
        = RaftProto.GetConfigurationResponse.newBuilder();
responseBuilder.setResCode(RaftProto.ResCode.RES_CODE_SUCCESS);

raftNode.getLock().lock();
try {
    ...
} finally {
    raftNode.getLock().unlock();
}

RaftProto.GetConfigurationResponse response = responseBuilder.build();
return response;
```

> 对获取集群配置信息的请求进行相应处理并构造响应。
>
> 处理逻辑使用同步锁保护。



```java
try {
    RaftProto.Configuration configuration = raftNode.getConfiguration();
    RaftProto.Server leader = ConfigurationUtils.getServer(configuration, raftNode.getLeaderId());
    responseBuilder.setLeader(leader);
    responseBuilder.addAllServers(configuration.getServersList());
} finally {
    ...
}
```

> 获取节点列表和Leader节点并响应。



##### addPeers()方法

```java
RaftProto.AddPeersResponse.Builder responseBuilder = RaftProto.AddPeersResponse.newBuilder();
responseBuilder.setResCode(RaftProto.ResCode.RES_CODE_FAIL);

...

RaftProto.AddPeersResponse response = responseBuilder.build();
return response;
```

> 对新增节点的请求进行相应处理并构造响应。



```java
if (request.getServersCount() == 0
        || request.getServersCount() % 2 != 0) {
    return responseBuilder.build();
}
for (RaftProto.Server server : request.getServersList()) {
    if (raftNode.getPeerMap().containsKey(server.getServerId())) {
        return responseBuilder.build();
    }
}
```

> 如果需要新增的节点数为0或为奇数，则需要新增的节点数不合法，进行失败响应。
>
> 遍历需要新增的节点，如果有节点的ID已经存在，则进行失败响应。



```java
List<Peer> requestPeers = new ArrayList<>(request.getServersCount());
for (RaftProto.Server server : request.getServersList()) {
    final Peer peer = new Peer(server);
    peer.setNextIndex(1);
    requestPeers.add(peer);
    raftNode.getPeerMap().putIfAbsent(server.getServerId(), peer);
    raftNode.getExecutorService().submit(new Runnable() {
        @Override
        public void run() {
            raftNode.appendEntries(peer);
        }
    });
}
```

> 遍历需要新增的节点，初始化日志复制的NextIndex，添加到节点列表，并提交日志复制任务到线程池，调用**RaftNode**类的**appendEntries()**方法，开始对新增节点进行日志同步。



```java
int catchUpNum = 0;
raftNode.getLock().lock();
try {
    while (catchUpNum < requestPeers.size()) {
        try {
            raftNode.getCatchUpCondition().await();
        } catch (InterruptedException ex) {
            ex.printStackTrace();
        }
        catchUpNum = 0;
        for (Peer peer : requestPeers) {
            if (peer.isCatchUp()) {
                catchUpNum++;
            }
        }
        if (catchUpNum == requestPeers.size()) {
            break;
        }
    }
} finally {
    raftNode.getLock().unlock();
}
```

> 进入循环，进行同步锁的条件等待，等待新增节点进行日志进度追赶，当一个新增节点完成追赶后会唤醒当前线程。遍历新增节点，检查是否完成追赶，当所有新增节点完成追赶后，退出循环。此过程使用同步锁保护。



```java
if (catchUpNum == requestPeers.size()) {
    raftNode.getLock().lock();
    byte[] configurationData;
    RaftProto.Configuration newConfiguration;
    try {
        newConfiguration = RaftProto.Configuration.newBuilder(raftNode.getConfiguration())
                .addAllServers(request.getServersList()).build();
        configurationData = newConfiguration.toByteArray();
    } finally {
        raftNode.getLock().unlock();
    }
    boolean success = raftNode.replicate(configurationData, RaftProto.EntryType.ENTRY_TYPE_CONFIGURATION);
    if (success) {
        responseBuilder.setResCode(RaftProto.ResCode.RES_CODE_SUCCESS);
    }
}
```

> 将新增节点后的集群配置信息封装为字节数据。此过程使用同步锁保护。
>
> 调用**RaftNode**类的**replicate()**方法，将新的集群配置数据进行集群写入。



```java
if (responseBuilder.getResCode() != RaftProto.ResCode.RES_CODE_SUCCESS) {
    raftNode.getLock().lock();
    try {
        for (Peer peer : requestPeers) {
            peer.getRpcClient().stop();
            raftNode.getPeerMap().remove(peer.getServer().getServerId());
        }
    } finally {
        raftNode.getLock().unlock();
    }
}
```

> 如果上一步骤中，新的集群配置数据写入不成功，则新增节点失败，需要遍历新增的节点，关闭RPC客户端，并从节点列表移除。此过程使用同步锁保护。



##### removePeers()方法

```java
RaftProto.RemovePeersResponse.Builder responseBuilder = RaftProto.RemovePeersResponse.newBuilder();
responseBuilder.setResCode(RaftProto.ResCode.RES_CODE_FAIL);

...

return responseBuilder.build();
```

> 对移除节点的请求进行相应处理并构造响应。



```java
if (request.getServersCount() == 0
    || request.getServersCount() % 2 != 0) {
    return responseBuilder.build();
}
raftNode.getLock().lock();
try {
    for (RaftProto.Server server : request.getServersList()) {
        if (!ConfigurationUtils.containsServer(raftNode.getConfiguration(), server.getServerId())) {
            return responseBuilder.build();
        }
    }
} finally {
    raftNode.getLock().unlock();
}
```

> 如果需要移除的节点数为0或为奇数，则需要移除的节点数不合法，进行失败响应。
>
> 遍历需要移除的节点，如果有节点的ID不存在，则进行失败响应。此过程使用同步锁保护。



```java
raftNode.getLock().lock();
RaftProto.Configuration newConfiguration;
byte[] configurationData;
try {
    newConfiguration = ConfigurationUtils.removeServers(
            raftNode.getConfiguration(), request.getServersList());
    configurationData = newConfiguration.toByteArray();
} finally {
    raftNode.getLock().unlock();
}
boolean success = raftNode.replicate(configurationData, RaftProto.EntryType.ENTRY_TYPE_CONFIGURATION);
if (success) {
    responseBuilder.setResCode(RaftProto.ResCode.RES_CODE_SUCCESS);
}
```

> 将移除节点后的集群配置信息封装为字节数据。此过程使用同步锁保护。
>
> 调用**RaftNode**类的**replicate()**方法，将新的集群配置数据进行集群写入。



#### 节点相互通信的服务RaftConsensusServiceImpl类

##### 继承自RaftConsensusService接口

```java
public interface RaftConsensusService {
    RaftProto.VoteResponse preVote(RaftProto.VoteRequest request);

    RaftProto.VoteResponse requestVote(RaftProto.VoteRequest request);

    RaftProto.AppendEntriesResponse appendEntries(RaftProto.AppendEntriesRequest request);

    RaftProto.InstallSnapshotResponse installSnapshot(RaftProto.InstallSnapshotRequest request);
}
```

> 定义了四个核心接口方法，分别为：处理预投票请求，处理请求投票请求，处理追加日志条目请求，处理复制快照请求。



##### 成员属性及初始化

```java
private RaftNode raftNode;

public RaftConsensusServiceImpl(RaftNode node) {
    this.raftNode = node;
}
```

> 初始化Raft节点。



##### preVote()方法

```java
raftNode.getLock().lock();
try {
    RaftProto.VoteResponse.Builder responseBuilder = RaftProto.VoteResponse.newBuilder();
    responseBuilder.setGranted(false);
    responseBuilder.setTerm(raftNode.getCurrentTerm());

    ...

    return responseBuilder.build();
} finally {
    raftNode.getLock().unlock();
}
```

> 对预投票请求进行相应处理并构造响应，响应时会携带当前节点的任期号。此过程使用同步锁保护。



```java
if (!ConfigurationUtils.containsServer(raftNode.getConfiguration(), request.getServerId())) {
    return responseBuilder.build();
}
if (request.getTerm() < raftNode.getCurrentTerm()) {
    return responseBuilder.build();
}
```

> 如果请求来源节点不在集群配置中，则直接进行失败响应。
>
> 如果请求来源节点的任期号小于当前节点的任期号，则直接进行失败响应。



```java
boolean isLogOk = request.getLastLogTerm() > raftNode.getLastLogTerm()
        || (request.getLastLogTerm() == raftNode.getLastLogTerm()
        && request.getLastLogIndex() >= raftNode.getRaftLog().getLastLogIndex());
if (!isLogOk) {
    return responseBuilder.build();
} else {
    responseBuilder.setGranted(true);
    responseBuilder.setTerm(raftNode.getCurrentTerm());
}
```

> 根据日志情况，判断当前节点是否能够投票给请求来源节点。请求来源节点的日志满足以下条件中的任意一个时，当前节点才能投票给请求来源节点：
>
> 1. 请求来源节点的最后一条日志的任期号，大于当前节点的最后一条日志的任期号。
> 2. 请求来源节点的最后一条日志的任期号，等于当前节点的最后一条日志的任期号，且请求来源节点的最后一条日志的索引，大于当前节点的最后一条日志的索引。
>
> 进行成功响应，并在响应中携带当前节点的任期号。



##### requestVote()方法

```java
raftNode.getLock().lock();
try {
    RaftProto.VoteResponse.Builder responseBuilder = RaftProto.VoteResponse.newBuilder();
    responseBuilder.setGranted(false);
    responseBuilder.setTerm(raftNode.getCurrentTerm());

    ...

    return responseBuilder.build();
} finally {
    raftNode.getLock().unlock();
}
```

> 对请求投票请求进行相应处理并构造响应，响应时会携带当前节点的任期号。此过程使用同步锁保护。



```java
if (!ConfigurationUtils.containsServer(raftNode.getConfiguration(), request.getServerId())) {
    return responseBuilder.build();
}
if (request.getTerm() < raftNode.getCurrentTerm()) {
    return responseBuilder.build();
}
```

> 如果请求来源节点不在集群配置中，则直接进行失败响应。
>
> 如果请求来源节点的任期号小于当前节点的任期号，则直接进行失败响应。



```java
if (request.getTerm() > raftNode.getCurrentTerm()) {
    raftNode.stepDown(request.getTerm());
}
```

> 如果请求来源节点的任期号大于当前节点的任期号，则当前节点需要调用**RaftNode**类的**stepDown()**方法，放弃竞选，更新自己的任期号。



```java
boolean logIsOk = request.getLastLogTerm() > raftNode.getLastLogTerm()
        || (request.getLastLogTerm() == raftNode.getLastLogTerm()
        && request.getLastLogIndex() >= raftNode.getRaftLog().getLastLogIndex());
if (raftNode.getVotedFor() == 0 && logIsOk) {
    raftNode.stepDown(request.getTerm());
    raftNode.setVotedFor(request.getServerId());
    raftNode.getRaftLog().updateMetaData(raftNode.getCurrentTerm(), raftNode.getVotedFor(), null, null);
    responseBuilder.setGranted(true);
    responseBuilder.setTerm(raftNode.getCurrentTerm());
}
```

> 根据日志情况，判断当前节点是否能够投票给请求来源节点。请求来源节点的日志满足以下条件中的任意一个时，当前节点才能投票给请求来源节点：
>
> 1. 请求来源节点的最后一条日志的任期号，大于当前节点的最后一条日志的任期号。
> 2. 请求来源节点的最后一条日志的任期号，等于当前节点的最后一条日志的任期号，且请求来源节点的最后一条日志的索引，大于当前节点的最后一条日志的索引。
>
> 如果当前节点能够投票给请求来源节点，而且当前节点在本轮选举中还没有投票给其他节点，则投票给请求来源节点。
>
> 当前节点调用**RaftNode**类的**stepDown()**方法，放弃竞选，更新自己的任期号。
>
> 进行成功响应，并在响应中携带当前节点的任期号。



##### appendEntries()方法

```java
raftNode.getLock().lock();
try {
    RaftProto.AppendEntriesResponse.Builder responseBuilder
            = RaftProto.AppendEntriesResponse.newBuilder();
    responseBuilder.setTerm(raftNode.getCurrentTerm());
    responseBuilder.setResCode(RaftProto.ResCode.RES_CODE_FAIL);
    responseBuilder.setLastLogIndex(raftNode.getRaftLog().getLastLogIndex());
    
    ...
    
    return responseBuilder.build();
} finally {
    raftNode.getLock().unlock();
}
```

> 对追加日志条目请求进行相应处理并构造响应，响应时会携带当前节点的任期号以及最后一个日志条目的索引。此过程使用同步锁保护。



```java
if (request.getTerm() < raftNode.getCurrentTerm()) {
    return responseBuilder.build();
}
```

> 如果请求来源节点的任期号小于当前节点的任期号，则直接进行失败响应。



```java
raftNode.stepDown(request.getTerm());
```

> 当前节点调用**RaftNode**类的**stepDown()**方法，更新自己的任期号。



```java
if (raftNode.getLeaderId() == 0) {
    raftNode.setLeaderId(request.getServerId());
}
```

> 如果当前节点维护的Leader的节点ID为0，表示还没有发现集群的Leader，则将请求来源节点设置为Leader。



```java
if (raftNode.getLeaderId() != request.getServerId()) {
    raftNode.stepDown(request.getTerm() + 1);
    responseBuilder.setResCode(RaftProto.ResCode.RES_CODE_FAIL);
    responseBuilder.setTerm(request.getTerm() + 1);
    return responseBuilder.build();
}
```

> 如果当前节点维护的Leader的节点ID与请求来源节点的ID不同，表明集群中出现了不止一个节点认为自己是Leader，此时需要增加当前以及需要响应的任期号，并进行失败响应，以触发下一轮选举。
>
> 当前节点调用**RaftNode**类的**stepDown()**方法，更新自己的任期号。



```java
if (request.getPrevLogIndex() > raftNode.getRaftLog().getLastLogIndex()) {
    return responseBuilder.build();
}
```

> 进行一致性检查。
>
> 如果追加日志条目的前一个条目的索引大于当前节点的最后一个条目的索引，说明存在日志空洞需要修复，直接进行失败响应。



```java
if (request.getPrevLogIndex() >= raftNode.getRaftLog().getFirstLogIndex()
        && raftNode.getRaftLog().getEntryTerm(request.getPrevLogIndex())
        != request.getPrevLogTerm()) {
    Validate.isTrue(request.getPrevLogIndex() > 0);
    responseBuilder.setLastLogIndex(request.getPrevLogIndex() - 1);
    return responseBuilder.build();
}
```

> 进行一致性检查。
>
> 如果追加日志条目的前一个条目的索引大于等于当前节点的第一个日志条目的索引，且追加日志条目的前一个条目的任期号与当前节点相同索引位置的日志条目的任期号不同，则进行失败响应，并要求Leader下次的追加日志条目请求发送前一个日志条目。



```java
if (request.getEntriesCount() == 0) {
    responseBuilder.setResCode(RaftProto.ResCode.RES_CODE_SUCCESS);
    responseBuilder.setTerm(raftNode.getCurrentTerm());
    responseBuilder.setLastLogIndex(raftNode.getRaftLog().getLastLogIndex());
    advanceCommitIndex(request);
    return responseBuilder.build();
}
```

> 如果请求中没有携带日志条目数据，则本次请求是一个No-op请求。
>
> 调用**advanceCommitIndex()**方法，对未提交的日志条目进行提交。
>
> 进行成功响应，并在响应中携带当前节点的任期号和最后一个日志条目的索引。



```java
responseBuilder.setResCode(RaftProto.ResCode.RES_CODE_SUCCESS);
List<RaftProto.LogEntry> entries = new ArrayList<>();
long index = request.getPrevLogIndex();
for (RaftProto.LogEntry entry : request.getEntriesList()) {
    index++;
    if (index < raftNode.getRaftLog().getFirstLogIndex()) {
        continue;
    }
    if (raftNode.getRaftLog().getLastLogIndex() >= index) {
        if (raftNode.getRaftLog().getEntryTerm(index) == entry.getTerm()) {
            continue;
        }
        long lastIndexKept = index - 1;
        raftNode.getRaftLog().truncateSuffix(lastIndexKept);
    }
    entries.add(entry);
}
raftNode.getRaftLog().append(entries);
responseBuilder.setLastLogIndex(raftNode.getRaftLog().getLastLogIndex());
advanceCommitIndex(request);
```

> 遍历请求中的日志条目列表。
>
> 跳过索引小于当前节点的第一个日志条目的索引的日志条目。
>
> 如果日志条目的索引小于等于当前节点的最后一个条目的索引，比较日志条目的任期号和当前节点该索引位置的日志条目的任期号，如果不相同，则发生日志冲突，需要调用**SegmentedLog**类的**truncateSuffix()**方法，将当前节点该索引位置及之后的日志条目全部删除，如果相同，则日志没有冲突，跳过此日志条目。
>
> 调用**SegmentedLog**类的**append()**方法，将没有被跳过的日志条目追加到当前节点的日志中。
>
> 调用**advanceCommitIndex()**方法，对未提交的日志条目进行提交。
>
> 进行成功响应，并在响应中携带当前节点追加日志条目后的最后一个条目的索引。



##### installSnapshot()方法

```java
RaftProto.InstallSnapshotResponse.Builder responseBuilder
        = RaftProto.InstallSnapshotResponse.newBuilder();
responseBuilder.setResCode(RaftProto.ResCode.RES_CODE_FAIL);

...

return responseBuilder.build();
```

> 对复制快照请求进行相应处理并构造响应。



```java
raftNode.getLock().lock();
try {
    responseBuilder.setTerm(raftNode.getCurrentTerm());
    if (request.getTerm() < raftNode.getCurrentTerm()) {
        return responseBuilder.build();
    }
    raftNode.stepDown(request.getTerm());
    if (raftNode.getLeaderId() == 0) {
        raftNode.setLeaderId(request.getServerId());
    }
} finally {
    raftNode.getLock().unlock();
}
```

> 如果请求来源节点的任期号小于当前节点的任期号，则直接进行失败响应，并在响应中携带当前节点的任期号。
>
> 当前节点调用**RaftNode**类的**stepDown()**方法，更新自己的任期号。
>
> 如果当前节点维护的Leader的节点ID为0，表示还没有发现集群的Leader，则将请求来源节点设置为Leader。
>
> 此过程使用同步锁保护。



```java
if (raftNode.getSnapshot().getIsTakeSnapshot().get()) {
    return responseBuilder.build();
}
```

> 判断当前节点是否正在创建快照，如果是，则直接进行失败响应。



```java
raftNode.getSnapshot().getIsInstallSnapshot().set(true);
RandomAccessFile randomAccessFile = null;
raftNode.getSnapshot().getLock().lock();
try {
    ...
} catch (IOException ex) {
    ...
} finally {
    RaftFileUtils.closeFile(randomAccessFile);
    raftNode.getSnapshot().getLock().unlock();
}
```

> 进入正在复制快照的状态，并从Leader复制快照。
>
> 复制快照文件的逻辑使用同步锁保护。



```java
try {
    String tmpSnapshotDir = raftNode.getSnapshot().getSnapshotDir() + ".tmp";
    File file = new File(tmpSnapshotDir);
    if (request.getIsFirst()) {
        if (file.exists()) {
            file.delete();
        }
        file.mkdir();
        raftNode.getSnapshot().updateMetaData(tmpSnapshotDir,
                request.getSnapshotMetaData().getLastIncludedIndex(),
                request.getSnapshotMetaData().getLastIncludedTerm(),
                request.getSnapshotMetaData().getConfiguration());
    }

    ...
} catch (IOException ex) {
    ...
} finally {
    RaftFileUtils.closeFile(randomAccessFile);
    raftNode.getSnapshot().getLock().unlock();
}
```

> 如果是复制快照的第一次通信，则创建快照文件临时存储路径的文件夹。



```java
try {
    ...

    String currentDataDirName = tmpSnapshotDir + File.separator + "data";
    File currentDataDir = new File(currentDataDirName);
    if (!currentDataDir.exists()) {
        currentDataDir.mkdirs();
    }
    String currentDataFileName = currentDataDirName + File.separator + request.getFileName();
    File currentDataFile = new File(currentDataFileName);
    if (!currentDataFile.getParentFile().exists()) {
        currentDataFile.getParentFile().mkdirs();
    }
    if (!currentDataFile.exists()) {
        currentDataFile.createNewFile();
    }

    ...
} catch (IOException ex) {
    ...
} finally {
    RaftFileUtils.closeFile(randomAccessFile);
    raftNode.getSnapshot().getLock().unlock();
}
```

> 创建快照文件存储路径的文件夹和文件。



```java
randomAccessFile = RaftFileUtils.openFile(
        tmpSnapshotDir + File.separator + "data",
        request.getFileName(), "rw");
randomAccessFile.seek(request.getOffset());
randomAccessFile.write(request.getData().toByteArray());
```

> 将请求中的快照文件数据写入磁盘文件。



```java
if (request.getIsLast()) {
    File snapshotDirFile = new File(raftNode.getSnapshot().getSnapshotDir());
    if (snapshotDirFile.exists()) {
        FileUtils.deleteDirectory(snapshotDirFile);
    }
    FileUtils.moveDirectory(new File(tmpSnapshotDir), snapshotDirFile);
}
responseBuilder.setResCode(RaftProto.ResCode.RES_CODE_SUCCESS);
```

> 如果是复制快照的最后一次通信，则使用临时存储路径的文件夹覆盖真实存储路径的文件夹。



```java
if (request.getIsLast() && responseBuilder.getResCode() == RaftProto.ResCode.RES_CODE_SUCCESS) {
    String snapshotDataDir = raftNode.getSnapshot().getSnapshotDir() + File.separator + "data";
    raftNode.getStateMachine().readSnapshot(snapshotDataDir);
    long lastSnapshotIndex;
    raftNode.getSnapshot().getLock().lock();
    try {
        raftNode.getSnapshot().reload();
        lastSnapshotIndex = raftNode.getSnapshot().getMetaData().getLastIncludedIndex();
    } finally {
        raftNode.getSnapshot().getLock().unlock();
    }

    raftNode.getLock().lock();
    try {
        raftNode.getRaftLog().truncatePrefix(lastSnapshotIndex + 1);
    } finally {
        raftNode.getLock().unlock();
    }
}

if (request.getIsLast()) {
    raftNode.getSnapshot().getIsInstallSnapshot().set(false);
}
```

> 如果是复制快照的最后一次通信，并且快照文件复制成功，则开始加载快照文件。
>
> 调用**StateMachine**接口的**readSnapshot()**方法，将快照文件数据应用到当前节点的复制状态机。
>
> 重新加载快照的元信息。此过程使用同步锁保护。
>
> 根据快照文件包含的最后一条日志索引，调用**SegmentedLog**类的**truncatePrefix()**方法，将当前节点该索引位置及之前的日志条目全部删除。此过程使用同步锁保护。
>
> 如果是复制快照的最后一次通信，退出正在复制快照的状态。



##### 推进提交进度的advanceCommitIndex()方法

```java
long newCommitIndex = Math.min(request.getCommitIndex(),
        request.getPrevLogIndex() + request.getEntriesCount());
raftNode.setCommitIndex(newCommitIndex);
raftNode.getRaftLog().updateMetaData(null,null, null, newCommitIndex);
```

> 计算允许提交的最大日志条目索引，值为以下二者中的较小者：
>
> 1. Leader的CommitIndex。
> 2. 追加日志条目的前一个条目的索引与本次请求追加的日志条目数量。
>
> 更新当前节点的CommitIndex。



```java
if (raftNode.getLastAppliedIndex() < raftNode.getCommitIndex()) {
    for (long index = raftNode.getLastAppliedIndex() + 1;
         index <= raftNode.getCommitIndex(); index++) {
        RaftProto.LogEntry entry = raftNode.getRaftLog().getEntry(index);
        if (entry != null) {
            if (entry.getType() == RaftProto.EntryType.ENTRY_TYPE_DATA) {
                raftNode.getStateMachine().apply(entry.getData().toByteArray());
            } else if (entry.getType() == RaftProto.EntryType.ENTRY_TYPE_CONFIGURATION) {
                raftNode.applyConfiguration(entry);
            }
        }
        raftNode.setLastAppliedIndex(index);
    }
}
```

> 如果当前节点最后应用到复制状态机的日志条目索引小于当前节点的CommitIndex，则遍历二者之间的日志条目。
>
> 如果是一般日志条目，调用**StateMachine**接口的**apply()**方法，应用到当前节点的复制状态机。
>
> 如果是集群配置日志条目，则调用**RaftNode**类的**applyConfiguration()**方法，更新当前节点维护的集群配置信息。
>
> 更新当前节点最后应用到复制状态机的日志条目索引。



#### Raft节点核心功能RaftNode类

##### 成员属性

```java
private RaftOptions raftOptions;
private RaftProto.Configuration configuration;
private ConcurrentMap<Integer, Peer> peerMap = new ConcurrentHashMap<>();
private RaftProto.Server localServer;
private StateMachine stateMachine;
private SegmentedLog raftLog;
private Snapshot snapshot;
```

> 维护了Raft运行参数、集群配置信息、节点状态信息集合、当前服务器信息、复制状态机、日志文件、快照文件。



```java
private NodeState state = NodeState.STATE_FOLLOWER;
private long currentTerm;
private int votedFor;
private int leaderId;
private long commitIndex;
private volatile long lastAppliedIndex;
```

> 维护了节点角色、任期号、选举时投票的目标节点ID、集群的Leader的节点ID、已提交的最后一个日志条目索引（CommitIndex）、已应用到复制状态机的最后一个日志条目索引。其中节点角色初始化为Follower。



```java
private Lock lock = new ReentrantLock();
private Condition commitIndexCondition = lock.newCondition();
private Condition catchUpCondition = lock.newCondition();
private ExecutorService executorService;
private ScheduledExecutorService scheduledExecutorService;
private ScheduledFuture electionScheduledFuture;
private ScheduledFuture heartbeatScheduledFuture;
```

> 维护了同步锁、提交进度的锁等待条件、日志进度追赶的锁等待条件、异步发送请求的线程池、选举超时、心跳超时和快照检查的定时任务线程池、选举超时定时任务、心跳超时定时任务。



##### 构造器

```java
this.raftOptions = raftOptions;
RaftProto.Configuration.Builder confBuilder = RaftProto.Configuration.newBuilder();
for (RaftProto.Server server : servers) {
    confBuilder.addServers(server);
}
configuration = confBuilder.build();
this.localServer = localServer;
this.stateMachine = stateMachine;
raftLog = new SegmentedLog(raftOptions.getDataDir(), raftOptions.getMaxSegmentFileSize());
snapshot = new Snapshot(raftOptions.getDataDir());
snapshot.reload();
```

> 初始化Raft运行参数、集群配置信息、当前服务器信息、复制状态机、日志文件、快照文件。



```java
currentTerm = raftLog.getMetaData().getCurrentTerm();
votedFor = raftLog.getMetaData().getVotedFor();
commitIndex = Math.max(snapshot.getMetaData().getLastIncludedIndex(), raftLog.getMetaData().getCommitIndex());
```

> 初始化任期号、选举时投票的目标节点ID、CommitIndex。



```java
if (snapshot.getMetaData().getLastIncludedIndex() > 0
        && raftLog.getFirstLogIndex() <= snapshot.getMetaData().getLastIncludedIndex()) {
    raftLog.truncatePrefix(snapshot.getMetaData().getLastIncludedIndex() + 1);
}
```

> 判断快照文件包含的最后一条日志索引是否大于0，且大于当前节点的第一个日志条目的索引的日志条目，如果是，则调用**SegmentedLog**类的**truncatePrefix()**方法，将当前节点该索引位置及之前的日志条目全部删除。



```java
RaftProto.Configuration snapshotConfiguration = snapshot.getMetaData().getConfiguration();
if (snapshotConfiguration.getServersCount() > 0) {
    configuration = snapshotConfiguration;
}
String snapshotDataDir = snapshot.getSnapshotDir() + File.separator + "data";
stateMachine.readSnapshot(snapshotDataDir);
```

> 加载快照文件的内容，读取保存的集群配置信息，并调用**StateMachine**接口的**readSnapshot()**方法，加载数据到复制状态机。



```java
for (long index = snapshot.getMetaData().getLastIncludedIndex() + 1;
     index <= commitIndex; index++) {
    RaftProto.LogEntry entry = raftLog.getEntry(index);
    if (entry.getType() == RaftProto.EntryType.ENTRY_TYPE_DATA) {
        stateMachine.apply(entry.getData().toByteArray());
    } else if (entry.getType() == RaftProto.EntryType.ENTRY_TYPE_CONFIGURATION) {
        applyConfiguration(entry);
    }
}
lastAppliedIndex = commitIndex;
```

> 如果当前节点最后应用到复制状态机的日志条目索引小于当前节点的CommitIndex，则遍历二者之间的日志条目。
>
> 如果是一般日志条目，调用**StateMachine**接口的**apply()**方法，应用到当前节点的复制状态机。
>
> 如果是集群配置日志条目，则调用**applyConfiguration()**方法，更新当前节点维护的集群配置信息。
>
> 将已应用到复制状态机的最后一个日志条目索引更新为CommitIndex，表明所有已提交的日志条目都被应用到复制状态机。



##### 初始化方法init()

```java
for (RaftProto.Server server : configuration.getServersList()) {
    if (!peerMap.containsKey(server.getServerId())
            && server.getServerId() != localServer.getServerId()) {
        Peer peer = new Peer(server);
        peer.setNextIndex(raftLog.getLastLogIndex() + 1);
        peerMap.put(server.getServerId(), peer);
    }
}
```

> 遍历集群配置信息中的每个节点，如果不是当前节点，且还未加入节点状态信息集合中，则加入到集合中，同时为每个其他节点维护NextIndex。



```java
executorService = new ThreadPoolExecutor(
        raftOptions.getRaftConsensusThreadNum(),
        raftOptions.getRaftConsensusThreadNum(),
        60,
        TimeUnit.SECONDS,
        new LinkedBlockingQueue<Runnable>());
scheduledExecutorService = Executors.newScheduledThreadPool(2);
```

> 初始化请求线程池和定时任务线程池。



```java
scheduledExecutorService.scheduleWithFixedDelay(new Runnable() {
    @Override
    public void run() {
        takeSnapshot();
    }
}, raftOptions.getSnapshotPeriodSeconds(), raftOptions.getSnapshotPeriodSeconds(), TimeUnit.SECONDS);
resetElectionTimer();
```

> 启动快照检查定时任务，超时后将调用**takeSnapshot()**方法检查是否需要创建快照，并提交到定时任务线程池。
>
> 调用**resetElectionTimer()**方法，重置选举超时计时器。



##### 创建快照的takeSnapshot()方法

```java
if (snapshot.getIsInstallSnapshot().get()) {
    return;
}
```

> 如果当前节点正在创建快照，则直接返回。



```java
snapshot.getIsTakeSnapshot().compareAndSet(false, true);
try {
	...
} finally {
    snapshot.getIsTakeSnapshot().compareAndSet(true, false);
}
```

> 进入正在创建快照的状态。快照文件创建完成后或出现异常时，退出正在创建快照的状态。



```java
try {
    long localLastAppliedIndex;
    long lastAppliedTerm = 0;
    RaftProto.Configuration.Builder localConfiguration = RaftProto.Configuration.newBuilder();
    lock.lock();
    try {
        if (raftLog.getTotalSize() < raftOptions.getSnapshotMinLogSize()) {
            return;
        }
        if (lastAppliedIndex <= snapshot.getMetaData().getLastIncludedIndex()) {
            return;
        }
        localLastAppliedIndex = lastAppliedIndex;
        if (lastAppliedIndex >= raftLog.getFirstLogIndex()
                && lastAppliedIndex <= raftLog.getLastLogIndex()) {
            lastAppliedTerm = raftLog.getEntryTerm(lastAppliedIndex);
        }
        localConfiguration.mergeFrom(configuration);
    } finally {
        lock.unlock();
    }

	...
} finally {
    ...
}
```

> 如果当前节点的日志文件的总大小没有达到创建快照的阈值，则直接返回。
>
> 如果当前节点的已应用到复制状态机的最后一个日志条目索引小于快照文件包含的最后一个日志条目的索引，表示快照文件已经包含所有已经应用到复制状态机的日志条目，直接返回。
>
> 如果当前节点的已应用到复制状态机的最后一个日志条目索引，大于等于日志文件的第一个日志条目的索引，并且小于等于日志文件的最后一个日志条目的索引，则获取已应用到复制状态机的最后一个日志条目的任期号。
>
> 获取集群配置信息。
>
> 此过程使用同步锁保护。



```java
try {
	...

    boolean success = false;
    snapshot.getLock().lock();
    try {
        String tmpSnapshotDir = snapshot.getSnapshotDir() + ".tmp";
        snapshot.updateMetaData(tmpSnapshotDir, localLastAppliedIndex,
                lastAppliedTerm, localConfiguration.build());
        String tmpSnapshotDataDir = tmpSnapshotDir + File.separator + "data";
        stateMachine.writeSnapshot(tmpSnapshotDataDir);
        try {
            File snapshotDirFile = new File(snapshot.getSnapshotDir());
            if (snapshotDirFile.exists()) {
                FileUtils.deleteDirectory(snapshotDirFile);
            }
            FileUtils.moveDirectory(new File(tmpSnapshotDir),
                    new File(snapshot.getSnapshotDir()));
            success = true;
        } catch (IOException ex) {
            ...
        }
    } finally {
        snapshot.getLock().unlock();
    }

    ...
} finally {
    ...
}
```

> 获取快照文件临时存储路径的文件夹。
>
> 更新快照文件的元信息，包括快照文件临时存储路径的文件夹、已应用到复制状态机的最后一个日志条目索引、已应用到复制状态机的最后一个日志条目的任期号、集群配置信息。
>
> 调用**StateMachine**接口的**writeSnapshot()**方法，创建快照文件并写入临时存储路径。
>
> 使用临时存储路径的文件夹覆盖真实存储路径的文件夹。
>
> 创建快照的结果为成功。
>
> 此过程使用同步锁保护。



```java
try {
	...

    if (success) {
        long lastSnapshotIndex = 0;
        snapshot.getLock().lock();
        try {
            snapshot.reload();
            lastSnapshotIndex = snapshot.getMetaData().getLastIncludedIndex();
        } finally {
            snapshot.getLock().unlock();
        }
        lock.lock();
        try {
            if (lastSnapshotIndex > 0 && raftLog.getFirstLogIndex() <= lastSnapshotIndex) {
                raftLog.truncatePrefix(lastSnapshotIndex + 1);
            }
        } finally {
            lock.unlock();
        }
    }
} finally {
    ...
}
```

>如果创建快照的结果为成功，重新加载快照文件，并获取快照文件包含的最后一个日志条目的索引，如果该索引大于0且小于等于日志文件的第一个日志条目的索引，则调用**SegmentedLog**类的**truncatePrefix()**方法，将当前节点该索引位置及之前的日志条目全部删除。
>
>上述两个步骤分别使用同步锁保护。



##### 领导者选举：重置选举超时计时器的resetElectionTimer()方法

```java
if (electionScheduledFuture != null && !electionScheduledFuture.isDone()) {
    electionScheduledFuture.cancel(true);
}
```

> 如果已经有选举超时任务在执行，并且还没执行完成，即没有发生选举超时，则取消已有的选举超时任务。



```java
electionScheduledFuture = scheduledExecutorService.schedule(new Runnable() {
    @Override
    public void run() {
        startPreVote();
    }
}, getElectionTimeoutMs(), TimeUnit.MILLISECONDS);
```

> 重新启动一个选举超时任务，超时后将调用**startPreVote()**方法发起预投票，提交到定时任务线程池。
>
> 调用**getElectionTimeoutMs()**方法获取选举超时时间。



##### 领导者选举：获取选举超时时间的getElectionTimeoutMs()方法

```java
ThreadLocalRandom random = ThreadLocalRandom.current();
int randomElectionTimeout = raftOptions.getElectionTimeoutMilliseconds()
        + random.nextInt(0, raftOptions.getElectionTimeoutMilliseconds());
return randomElectionTimeout;
```

> 根据配置的选举超时时间，生成一个范围在1到2倍之间的随机的选举超时时间。这是减少选举冲突的优化方式。



##### 领导者选举：发起预投票的startPreVote()方法

```java
lock.lock();
try {
    if (!ConfigurationUtils.containsServer(configuration, localServer.getServerId())) {
        resetElectionTimer();
        return;
    }
    state = NodeState.STATE_PRE_CANDIDATE;
} finally {
    lock.unlock();
}
```

> 如果当前节点已经不在集群中，则重置选举超时计时器并直接返回。
>
> 转换为预备Candidate角色。
>
> 此过程使用同步锁保护。



```java
for (RaftProto.Server server : configuration.getServersList()) {
    if (server.getServerId() == localServer.getServerId()) {
        continue;
    }
    final Peer peer = peerMap.get(server.getServerId());
    executorService.submit(new Runnable() {
        @Override
        public void run() {
            preVote(peer);
        }
    });
}
resetElectionTimer();
```

> 遍历集群中的所有节点，跳过当前节点。提交任务到请求线程池进行异步处理，调用**preVote()**方法，向其他所有节点发送预投票请求。
>
> 重置选举超时计时器。



##### 领导者选举：发起预投票请求的preVote()方法

```java
RaftProto.VoteRequest.Builder requestBuilder = RaftProto.VoteRequest.newBuilder();
lock.lock();
try {
    peer.setVoteGranted(null);
    requestBuilder.setServerId(localServer.getServerId())
            .setTerm(currentTerm)
            .setLastLogIndex(raftLog.getLastLogIndex())
            .setLastLogTerm(getLastLogTerm());
} finally {
    lock.unlock();
}

RaftProto.VoteRequest request = requestBuilder.build();
peer.getRaftConsensusServiceAsync().preVote(
        request, new PreVoteResponseCallback(peer, request));
```

> 构造预投票请求，在请求中携带当前节点的ID、任期号、最后一个日志条目的索引和最后一个日志条目的任期号。此过程使用同步锁保护。
>
> 获取其他节点的节点相互通信的服务接口**RaftConsensusService**，调用其**preVote()**方法，发送预投票请求进行处理。此处使用了异步响应的处理方式，需要额外传入回调接口**PreVoteResponseCallback**类。



##### 领导者选举：预投票请求异步响应回调接口内部类PreVoteResponseCallback的success()方法

```java
lock.lock();
try {
    peer.setVoteGranted(response.getGranted());
    if (currentTerm != request.getTerm() || state != NodeState.STATE_PRE_CANDIDATE) {
        return;
    }
    
    ...
} finally {
    lock.unlock();
}
```

> 从响应中获取投票结果并记录到节点状态信息中。
>
> 如果当前任期号与请求中的任期号不同，或当前节点不是预备Candidate角色，则现在已经不在本次响应的有效期内，本次响应不再需要处理。
>
> 整个处理逻辑使用同步锁保护。



```java
try {
    ...
    
    if (response.getTerm() > currentTerm) {
        stepDown(response.getTerm());
    } else {
        ...
    }
} finally {
    ...
}
```

> 如果响应中携带的任期号大于当前节点的任期号，则当前节点调用**stepDown()**方法，放弃竞选，更新自己的任期号。



```java
try {
    ...

    if (response.getTerm() > currentTerm) {
        ...
    } else {
        if (response.getGranted()) {
            int voteGrantedNum = 1;
            for (RaftProto.Server server : configuration.getServersList()) {
                if (server.getServerId() == localServer.getServerId()) {
                    continue;
                }
                Peer peer1 = peerMap.get(server.getServerId());
                if (peer1.isVoteGranted() != null && peer1.isVoteGranted() == true) {
                    voteGrantedNum += 1;
                }
            }
            if (voteGrantedNum > configuration.getServersCount() / 2) {
                startVote();
            }
        } else {
            ...
        }
    }
} finally {
    ...
}
```

> 如果在响应中获得了投票，则遍历集群中的所有节点，统计投票给当前节点的节点数，当前节点默认投票给自己。
>
> 如果超过集群半数节点投票给当前节点，则调用**startVote()**方法，发起投票。



##### 领导者选举：预投票请求异步响应回调接口内部类PreVoteResponseCallback的fail()方法

```java
peer.setVoteGranted(new Boolean(false));
```

> 视作没有获得投票，并记录到节点状态信息中。



##### 领导者选举：发起请求投票的startVote()方法

```java
lock.lock();
try {
    if (!ConfigurationUtils.containsServer(configuration, localServer.getServerId())) {
        resetElectionTimer();
        return;
    }
    currentTerm++;
    state = NodeState.STATE_CANDIDATE;
    leaderId = 0;
    votedFor = localServer.getServerId();
} finally {
    lock.unlock();
}
```

> 如果当前节点已经不在集群中，则重置选举超时计时器并直接返回。
>
> 增加任期号，转换为Candidate角色，重置维护的Leader的节点ID，并投票给自己。
>
> 此过程使用同步锁保护。



```java
for (RaftProto.Server server : configuration.getServersList()) {
    if (server.getServerId() == localServer.getServerId()) {
        continue;
    }
    final Peer peer = peerMap.get(server.getServerId());
    executorService.submit(new Runnable() {
        @Override
        public void run() {
            requestVote(peer);
        }
    });
}
```

> 遍历集群中的所有节点，跳过当前节点。提交任务到请求线程池进行异步处理，调用**requestVote()**方法，向其他所有节点发送请求投票请求。



##### 领导者选举：发起请求投票请求的requestVote()方法

```java
RaftProto.VoteRequest.Builder requestBuilder = RaftProto.VoteRequest.newBuilder();
lock.lock();
try {
    peer.setVoteGranted(null);
    requestBuilder.setServerId(localServer.getServerId())
            .setTerm(currentTerm)
            .setLastLogIndex(raftLog.getLastLogIndex())
            .setLastLogTerm(getLastLogTerm());
} finally {
    lock.unlock();
}

RaftProto.VoteRequest request = requestBuilder.build();
peer.getRaftConsensusServiceAsync().requestVote(
        request, new VoteResponseCallback(peer, request));
```

> 构造请求投票请求，在请求中携带当前节点的ID、任期号、最后一个日志条目的索引和最后一个日志条目的任期号。此过程使用同步锁保护。
>
> 获取其他节点的节点相互通信的服务接口**RaftConsensusService**，调用其**requestVote()**方法，发送请求投票请求进行处理。此处使用了异步响应的处理方式，需要额外传入回调接口**VoteResponseCallback**类。



##### 领导者选举：请求投票请求异步响应回调接口内部类VoteResponseCallback的success()方法

```java
lock.lock();
try {
    peer.setVoteGranted(response.getGranted());
    if (currentTerm != request.getTerm() || state != NodeState.STATE_CANDIDATE) {
        return;
    }

    ...
} finally {
    lock.unlock();
}
```

> 从响应中获取投票结果并记录到节点状态信息中。
>
> 如果当前任期号与请求中的任期号不同，或当前节点不是Candidate角色，则现在已经不在本次响应的有效期内，本次响应不再需要处理。
>
> 整个处理逻辑使用同步锁保护。



```java
try {
    ...

    if (response.getTerm() > currentTerm) {
        stepDown(response.getTerm());
    } else {
        ...
    }
} finally {
    ...
}
```

> 如果响应中携带的任期号大于当前节点的任期号，则当前节点调用**stepDown()**方法，放弃竞选，更新自己的任期号。



```java

try {
    ...

    if (response.getTerm() > currentTerm) {
        ...
    } else {
        if (response.getGranted()) {
            int voteGrantedNum = 0;
            if (votedFor == localServer.getServerId()) {
                voteGrantedNum += 1;
            }
            for (RaftProto.Server server : configuration.getServersList()) {
                if (server.getServerId() == localServer.getServerId()) {
                    continue;
                }
                Peer peer1 = peerMap.get(server.getServerId());
                if (peer1.isVoteGranted() != null && peer1.isVoteGranted() == true) {
                    voteGrantedNum += 1;
                }
            }
            if (voteGrantedNum > configuration.getServersCount() / 2) {
                becomeLeader();
            }
        } else {
            ...
        }
    }
} finally {
    ...
}
```

> 如果在响应中获得了投票，则遍历集群中的所有节点，统计投票给当前节点的节点数，当前节点默认投票给自己。
>
> 如果超过集群半数节点投票给当前节点，则调用**becomeLeader()**方法，成为Leader。



##### 领导者选举：请求投票请求异步响应回调接口内部类VoteResponseCallback的fail()方法

```java
peer.setVoteGranted(new Boolean(false));
```

> 视作没有获得投票，并记录到节点状态信息中。



##### 领导者选举：晋升Leader的becomeLeader()方法

```java
state = NodeState.STATE_LEADER;
leaderId = localServer.getServerId();
```

> 转换为Leader角色，将维护的Leader的节点ID设置为当前节点的ID。



```java
if (electionScheduledFuture != null && !electionScheduledFuture.isDone()) {
    electionScheduledFuture.cancel(true);
}
startNewHeartbeat();
```

> 如果已经有选举超时任务在执行，并且还没执行完成，即没有发生选举超时，则取消已有的选举超时任务。
>
> 调用**startNewHeartbeat()**方法，开始向其他节点发送心跳。



##### 领导者选举：Leader发送心跳的startNewHeartbeat()方法

```java
for (final Peer peer : peerMap.values()) {
    executorService.submit(new Runnable() {
        @Override
        public void run() {
            appendEntries(peer);
        }
    });
}
resetHeartbeatTimer();
```

> 遍历集群中的所有节点。提交任务到请求线程池进行异步处理，调用**appendEntries()**方法，向其他所有节点发送追加日志条目请求。
>
> 调用**resetHeartbeatTimer()**方法，启动心跳超时任务。



##### 领导者选举：重置心跳超时计时器的resetHeartbeatTimer()方法

```java
if (heartbeatScheduledFuture != null && !heartbeatScheduledFuture.isDone()) {
    heartbeatScheduledFuture.cancel(true);
}
```

> 如果已经有心跳超时任务在执行，并且还没执行完成，即没有发生心跳超时，则取消已有的心跳超时任务。



```java
heartbeatScheduledFuture = scheduledExecutorService.schedule(new Runnable() {
    @Override
    public void run() {
        startNewHeartbeat();
    }
}, raftOptions.getHeartbeatPeriodMilliseconds(), TimeUnit.MILLISECONDS);
```

> 重新启动一个心跳超时任务，超时后将调用**startNewHeartbeat()**方法发送心跳，提交到定时任务线程池。



##### 领导者选举：放弃竞选或退位并更新任期号的stepDown()方法

```java
if (currentTerm > newTerm) {
    return;
}
```

> 如果需要更新的任期号小于当前任期号，则无需进行任何处理，直接返回。



```java
if (currentTerm < newTerm) {
    currentTerm = newTerm;
    leaderId = 0;
    votedFor = 0;
    raftLog.updateMetaData(currentTerm, votedFor, null, null);
}
```

> 如果需要更新的任期号大于当前任期号，则更新当前任期号，并重置维护的Leader的节点ID和选举时投票的目标节点ID，等待参与选举投票或新Leader的心跳。



```java
state = NodeState.STATE_FOLLOWER;
if (heartbeatScheduledFuture != null && !heartbeatScheduledFuture.isDone()) {
    heartbeatScheduledFuture.cancel(true);
}
resetElectionTimer();
```

> 转换为Follower角色。
>
> 如果已经有心跳超时任务在执行，并且还没执行完成，即没有发生心跳超时，则取消已有的心跳超时任务。
>
> 调用**resetElectionTimer()**方法，重置选举超时计时器。



##### 日志复制：响应客户端进行集群写入的replicate()方法

```java
lock.lock();
long newLastLogIndex = 0;
try {
	...
} catch (Exception ex) {
	...
} finally {
	lock.unlock();
}
if (lastAppliedIndex < newLastLogIndex) {
	return false;
}
return true;
```

> 处理客户端的写请求以及集群配置变更请求时，需要将日志条目进行集群写入。
>
> 如果当前节点在写入后，最后一个日志条目的索引大于已应用到复制状态机的最后一个日志条目索引，则返回失败，否则返回成功。
>
> 集群写入的逻辑使用同步锁保护。



```java
try {
	if (state != NodeState.STATE_LEADER) {
    	return false;
	}
	
	...
} catch (Exception ex) {
	...
} finally {
	...
}
```

> 如果当前节点不是Leader节点，则不能处理写请求，直接返回失败。



```java
try {
	...

	RaftProto.LogEntry logEntry = RaftProto.LogEntry.newBuilder()
            .setTerm(currentTerm)
            .setType(entryType)
            .setData(ByteString.copyFrom(data)).build();
    List<RaftProto.LogEntry> entries = new ArrayList<>();
    entries.add(logEntry);
    newLastLogIndex = raftLog.append(entries);

	...
} catch (Exception ex) {
	...
} finally {
	...
}
```

> 将请求中的数据，连同当前任期号和日志类型，封装为日志条目。
>
> 调用**SegmentedLog**类的**append()**方法，追加日志条目到当前节点的日志文件中，更新最后一个日志条目的索引。



```java
try {
    ...
 
	for (RaftProto.Server server : configuration.getServersList()) {
        final Peer peer = peerMap.get(server.getServerId());
        executorService.submit(new Runnable() {
            @Override
            public void run() {
                appendEntries(peer);
            }
        });
    }

	...
} catch (Exception ex) {
	...
} finally {
	...
}
```

> 遍历集群中的所有节点。提交任务到请求线程池进行异步处理，调用**appendEntries()**方法，向其他所有节点发送追加日志条目请求。



```java
try {
    ...

	if (raftOptions.isAsyncWrite()) {
        return true;
    }

	...
} catch (Exception ex) {
	...
} finally {
	...
}
```

> 如果采用了异步复制策略，由于此时已经成功写入当前节点，则直接返回成功。



```java
try {
    ...

	long startTime = System.currentTimeMillis();
    while (lastAppliedIndex < newLastLogIndex) {
        if (System.currentTimeMillis() - startTime >= raftOptions.getMaxAwaitTimeout()) {
            break;
        }
        commitIndexCondition.await(raftOptions.getMaxAwaitTimeout(), TimeUnit.MILLISECONDS);
    }
} catch (Exception ex) {
	...
} finally {
	...
}
```

> 如果采用了同步复制策略，则进行有限时长的条件等待，等待日志条目同步成功并在当前节点应用到复制状态机，直到本次请求追加的日志条目都被应用到复制状态机或等待超时。



##### 日志复制：进行日志同步的appendEntries()方法

```java
RaftProto.AppendEntriesRequest.Builder requestBuilder = RaftProto.AppendEntriesRequest.newBuilder();
lock.lock();
try {
    long firstLogIndex = raftLog.getFirstLogIndex();
    if (peer.getNextIndex() < firstLogIndex) {
        isNeedInstallSnapshot = true;
    }
} finally {
    lock.unlock();
}
```

> 准备构造追加日志条目请求。
>
> 如果目标节点的NextIndex小于当前节点的第一个日志条目的索引，则需要先复制快照。此过程使用同步锁保护。



```java
if (isNeedInstallSnapshot) {
    if (!installSnapshot(peer)) {
        return;
    }
}
```

> 如果需要复制快照，则调用**installSnapshot()**方法，将快照文件复制到目标节点。如果复制快照成功，则继续进行日志同步，否则直接返回。



```java
long lastSnapshotIndex;
long lastSnapshotTerm;
snapshot.getLock().lock();
try {
    lastSnapshotIndex = snapshot.getMetaData().getLastIncludedIndex();
    lastSnapshotTerm = snapshot.getMetaData().getLastIncludedTerm();
} finally {
    snapshot.getLock().unlock();
}
```

> 获取当前节点的快照文件包含的最后一个日志条目的索引和任期号。此过程使用同步锁保护。



```java
lock.lock();
try {
    long firstLogIndex = raftLog.getFirstLogIndex();
    Validate.isTrue(peer.getNextIndex() >= firstLogIndex);
    prevLogIndex = peer.getNextIndex() - 1;
    long prevLogTerm;
    if (prevLogIndex == 0) {
        prevLogTerm = 0;
    } else if (prevLogIndex == lastSnapshotIndex) {
        prevLogTerm = lastSnapshotTerm;
    } else {
        prevLogTerm = raftLog.getEntryTerm(prevLogIndex);
    }
    requestBuilder.setServerId(localServer.getServerId());
    requestBuilder.setTerm(currentTerm);
    requestBuilder.setPrevLogTerm(prevLogTerm);
    requestBuilder.setPrevLogIndex(prevLogIndex);
    numEntries = packEntries(peer.getNextIndex(), requestBuilder);
    requestBuilder.setCommitIndex(Math.min(commitIndex, prevLogIndex + numEntries));
} finally {
    lock.unlock();
}
```

> 构造追加日志条目请求，在请求中携带当前节点的ID、任期号、上一个日志条目的索引和任期号、CommitIndex。
>
> 调用**packEntries()**方法，在请求中封装日志条目数据。
>
> 上一个日志条目的索引和任期号取决于以下逻辑：
>
> 1. 上一个日志条目的索引为目标节点的NextIndex减一。
> 2. 如果上一个日志条目的索引为0，表示正在复制第一个日志条目，则上一个日志条目的任期号为0.
> 3. 如果上一个日志条目的索引为当前节点的快照文件包含的最后一个日志条目的索引，则上一个日志条目的任期号为当前节点的快照文件包含的最后一个日志条目的任期号。
> 4. 如果上一个日志条目存在且没有被快照文件压缩，则查找日志文件，根据上一个日志条目的索引获取任期号。
>
> CommitIndex值为以下二者中的较小者：
>
> 1. 当前节点的的CommitIndex。
> 2. 追加日志条目的前一个条目的索引与本次请求追加的日志条目数量。
>
> 构造追加日志条目请求的逻辑使用同步锁保护。



```java
RaftProto.AppendEntriesRequest request = requestBuilder.build();
RaftProto.AppendEntriesResponse response = peer.getRaftConsensusServiceAsync().appendEntries(request);
```

> 构造追加日志条目请求。
>
> 获取其他节点的节点相互通信的服务接口**RaftConsensusService**，调用其**appendEntries()**方法，发送追加日志条目请求进行处理。此处使用了同步响应的处理方式，获取响应结果。



```java
lock.lock();
try {
    ...
} finally {
    lock.unlock();
}
```

> 对追加日志条目请求的响应结果进行相应处理。此过程使用同步锁保护。



```java
try {
    if (response == null) {
        if (!ConfigurationUtils.containsServer(configuration, peer.getServer().getServerId())) {
            peerMap.remove(peer.getServer().getServerId());
            peer.getRpcClient().stop();
        }
        return;
    }

    ...
} finally {
    ...
}
```

> 如果响应为空，则直接返回。
>
> 如果目标节点已经不在集群中，则从节点状态信息集合中移除该节点，并关闭该节点的RPC客户端。



```java
try {
    ...

    if (response.getTerm() > currentTerm) {
        stepDown(response.getTerm());
    } else {
        ...
    }
} finally {
    ...
}
```

> 如果响应中携带的任期号大于当前节点的任期号，则当前节点调用**stepDown()**方法，退位并更新自己的任期号。



```
try {
    ...

    if (response.getTerm() > currentTerm) {
        ...
    } else {
		if (response.getResCode() == RaftProto.ResCode.RES_CODE_SUCCESS) {
        	peer.setMatchIndex(prevLogIndex + numEntries);
        	peer.setNextIndex(peer.getMatchIndex() + 1);
        	if (ConfigurationUtils.containsServer(configuration, peer.getServer().getServerId())) {
        		advanceCommitIndex();
        	} else {
        		if (raftLog.getLastLogIndex() - peer.getMatchIndex() <= raftOptions.getCatchupMargin()) {
                	peer.setCatchUp(true);
                    catchUpCondition.signalAll();
                }
            }
		} else {
			...
		}
    }
} finally {
    ...
}
```

> 如果响应结果为成功，则更新目标节点的同步进度，并设置目标节点的NextIndex为同步进度加一。
>
> 如果目标节点是在集群中的运行的节点，则调用**advanceCommitIndex()**方法，推进当前节点的提交进度。
>
> 如果目标节点正在进行日志进度追赶，并且落后的进度小于阈值，则视为追赶完成，唤醒在同步锁的等待条件中等待的所有客户端线程。



```java
try {
    ...

    if (response.getTerm() > currentTerm) {
        ...
    } else {
		if (response.getResCode() == RaftProto.ResCode.RES_CODE_SUCCESS) {
        	...
		} else {
			peer.setNextIndex(response.getLastLogIndex() + 1);
		}
    }
} finally {
    ...
}
```

> 如果响应结果为不成功，则设置目标节点的NextIndex为响应中携带的最后一个日志条目的索引加一。



##### 日志复制：封装日志条目数据到追加日志条目请求的packEntries()方法

```java
long lastIndex = Math.min(raftLog.getLastLogIndex(),
        nextIndex + raftOptions.getMaxLogEntriesPerRequest() - 1);
for (long index = nextIndex; index <= lastIndex; index++) {
    RaftProto.LogEntry entry = raftLog.getEntry(index);
    requestBuilder.addEntries(entry);
}
return lastIndex - nextIndex + 1;
```

> 计算本次请求可以发送的最后一个日志条目的索引，值为以下二者中的较小者：
>
> 1. 当前节点的最后一个日志条目的索引。
> 2. 目标节点的NextIndex，加每次请求可以发送的最大日志条目数，减一。即按每次请求可以发送的最大日志条目数进行发送。
>
> 遍历目标节点的NextIndex到本次请求可以发送的最后一个日志条目的索引之间的日志条目，封装到请求中。
>
> 返回封装的日志条目数。



##### 日志复制：复制快照的installSnapshot()方法

```java
if (snapshot.getIsTakeSnapshot().get()) {
    return false;
}
if (!snapshot.getIsInstallSnapshot().compareAndSet(false, true)) {
    return false;
}
```

> 如果当前节点正在创建快照，则直接返回。
>
> 尝试进入正在复制快照的状态，如果失败，表示已经在进行快照文件的复制，直接返回。



```java
boolean isSuccess = true;
TreeMap<String, Snapshot.SnapshotDataFile> snapshotDataFileMap = snapshot.openSnapshotDataFiles();
try {
    ...
} finally {
    snapshot.closeSnapshotDataFiles(snapshotDataFileMap);
    snapshot.getIsInstallSnapshot().compareAndSet(true, false);
}
return isSuccess;
```

> 打开快照文件，分多次请求传输快照文件的数据。
>
> 复制完成后关闭快照文件，退出正在复制快照的状态，返回复制快照的结果。



```java
try {
    boolean isLastRequest = false;
    String lastFileName = null;
    long lastOffset = 0;
    long lastLength = 0;
    while (!isLastRequest) {
        RaftProto.InstallSnapshotRequest request
                = buildInstallSnapshotRequest(snapshotDataFileMap, lastFileName, lastOffset, lastLength);
        if (request == null) {
            isSuccess = false;
            break;
        }
        if (request.getIsLast()) {
            isLastRequest = true;
        }
        RaftProto.InstallSnapshotResponse response
                = peer.getRaftConsensusServiceAsync().installSnapshot(request);
        if (response != null && response.getResCode() == RaftProto.ResCode.RES_CODE_SUCCESS) {
            lastFileName = request.getFileName();
            lastOffset = request.getOffset();
            lastLength = request.getData().size();
        } else {
            isSuccess = false;
            break;
        }
    }

    ...
} finally {
    ...
}
```

> 进入循环，分多次请求发送快照文件数据，直到最后一次请求发送完成或其中一次请求没有得到成功响应。
>
> 调用**buildInstallSnapshotRequest()**方法，构造复制快照请求。
>
> 如果构造的请求为空，则复制快照的结果为失败，并退出循环。
>
> 如果构造的请求是本次复制快照的最后一次请求，则在执行完后退出循环。
>
> 获取其他节点的节点相互通信的服务接口**RaftConsensusService**，调用其**installSnapshot()**方法，发送复制快照请求进行处理。此处使用了同步响应的处理方式，获取响应结果。
>
> 如果响应不为空且为成功响应，则记录本次请求的快照文件名称、复制起始偏移量和复制数据的长度，用于构造下一次请求，否则复制快照的结果为失败，并退出循环。



```java
try {
    ...

    if (isSuccess) {
        long lastIncludedIndexInSnapshot;
        snapshot.getLock().lock();
        try {
            lastIncludedIndexInSnapshot = snapshot.getMetaData().getLastIncludedIndex();
        } finally {
            snapshot.getLock().unlock();
        }

        lock.lock();
        try {
            peer.setNextIndex(lastIncludedIndexInSnapshot + 1);
        } finally {
            lock.unlock();
        }
    }
} finally {
    ...
}
```

> 如果复制快照的结果为成功，则获取当前节点的快照文件包含的最后一个日志条目的索引，并设置目标节点的NextIndex为该索引加一。
>
> 上述两个步骤分别使用同步锁保护。



##### 日志复制：构造复制快照请求的buildInstallSnapshotRequest()方法

```java
RaftProto.InstallSnapshotRequest.Builder requestBuilder = RaftProto.InstallSnapshotRequest.newBuilder();
snapshot.getLock().lock();
try {
    ...
} catch (Exception ex) {
    return null;
} finally {
    snapshot.getLock().unlock();
}
lock.lock();
try {
    requestBuilder.setTerm(currentTerm);
    requestBuilder.setServerId(localServer.getServerId());
} finally {
    lock.unlock();
}
return requestBuilder.build();
```

> 根据上一次请求的快照文件名称、复制起始偏移量和复制数据长度，构造复制快照请求，如果出现异常则返回空。此过程使用同步锁保护。
>
> 在请求中携带当前节点的任期号和节点ID。此过程使用同步锁保护。



```java
try {
    if (lastFileName == null) {
        lastFileName = snapshotDataFileMap.firstKey();
        lastOffset = 0;
        lastLength = 0;
	}

    ...
} catch (Exception ex) {
    ...
} finally {
    ...
}
```

> 如果没有指定上一次请求的快照文件，表示正在构造第一次请求，则从第一个快照文件的开头开始复制。



```java
try {
    ...

    Snapshot.SnapshotDataFile lastFile = snapshotDataFileMap.get(lastFileName);
    long lastFileLength = lastFile.randomAccessFile.length();
    String currentFileName = lastFileName;
    long currentOffset = lastOffset + lastLength;
    int currentDataSize = raftOptions.getMaxSnapshotBytesPerRequest();
    Snapshot.SnapshotDataFile currentDataFile = lastFile;
	if (lastOffset + lastLength < lastFileLength) {
        if (lastOffset + lastLength + raftOptions.getMaxSnapshotBytesPerRequest() > lastFileLength) {
            currentDataSize = (int) (lastFileLength - (lastOffset + lastLength));
        }
    } else {
        ...
    }

    ...
} catch (Exception ex) {
    ...
} finally {
    ...
}
```

> 计算本次请求的复制起始偏移量，为上一次请求的复制起始偏移量加复制数据长度。
>
> 如果本次请求的复制起始偏移量小于上一次请求的快照文件长度，表示上一次请求没有将快照文件完整复制完成。
>
> 本次请求默认按照每次请求可以发送的最大快照文件数据长度进行发送，如果快照文件剩余的数据长度小于该长度，则本次请求仅发送快照文件剩余的数据。



```java
try {
    ...

	if (lastOffset + lastLength < lastFileLength) {
        ...
    } else {
        Map.Entry<String, Snapshot.SnapshotDataFile> currentEntry
        		= snapshotDataFileMap.higherEntry(lastFileName);
        if (currentEntry == null) {
        	return null;
        }
        currentDataFile = currentEntry.getValue();
        currentFileName = currentEntry.getKey();
        currentOffset = 0;
        int currentFileLenght = (int) currentEntry.getValue().randomAccessFile.length();
        if (currentFileLenght < raftOptions.getMaxSnapshotBytesPerRequest()) {
        	currentDataSize = currentFileLenght;
        }
    }

    ...
} catch (Exception ex) {
    ...
} finally {
    ...
}
```

> 如果本次请求的复制起始偏移量大于等于上一次请求的快照文件长度，表示上一次请求已经将快照文件完整复制完成。
>
> 获取下一个快照文件，如果获取不到，表示没有更多的快照文件需要复制，直接返回空。
>
> 从此快照文件的开头开始复制，如果此快照文件的长度小于每次请求可以发送的最大快照文件数据长度，则本次请求仅发送此快照文件的数据。



```java
try {
    ...

	byte[] currentData = new byte[currentDataSize];
	currentDataFile.randomAccessFile.seek(currentOffset);
	currentDataFile.randomAccessFile.read(currentData);

    ...
} catch (Exception ex) {
    ...
} finally {
    ...
}
```

> 根据本次请求发送的数据长度，从快照文件中读取相应的字节数据。



```java
try {
    ...

	requestBuilder.setData(ByteString.copyFrom(currentData));
    requestBuilder.setFileName(currentFileName);
    requestBuilder.setOffset(currentOffset);
    requestBuilder.setIsFirst(false);
    if (currentFileName.equals(snapshotDataFileMap.lastKey())
            && currentOffset + currentDataSize >= currentDataFile.randomAccessFile.length()) {
        requestBuilder.setIsLast(true);
    } else {
        requestBuilder.setIsLast(false);
    }
    if (currentFileName.equals(snapshotDataFileMap.firstKey()) && currentOffset == 0) {
        requestBuilder.setIsFirst(true);
        requestBuilder.setSnapshotMetaData(snapshot.getMetaData());
    } else {
        requestBuilder.setIsFirst(false);
    }
} catch (Exception ex) {
    ...
} finally {
    ...
}
```

> 封装请求，包括快照文件数据、快照文件名称、复制起始偏移量。
>
> 如果本次请求复制的快照文件是最后一个快照文件，并且复制起始偏移量和复制数据长度大于等于快照文件的长度，则标记本次请求为本次复制快照的最后一次请求。
>
> 如果本次请求复制的快照文件是第一个快照文件，并且复制起始偏移量是文件的开头，则标记本次请求为本次复制快照的第一次请求，并携带快照文件的元信息。



##### 日志复制：推进提交进度的advanceCommitIndex()方法

```java
int peerNum = configuration.getServersList().size();
long[] matchIndexes = new long[peerNum];
int i = 0;
for (RaftProto.Server server : configuration.getServersList()) {
    if (server.getServerId() != localServer.getServerId()) {
        Peer peer = peerMap.get(server.getServerId());
        matchIndexes[i++] = peer.getMatchIndex();
    }
}
matchIndexes[i] = raftLog.getLastLogIndex();
Arrays.sort(matchIndexes);
long newCommitIndex = matchIndexes[peerNum / 2];
```

> 遍历集群中的所有节点，收集所有Follower的同步进度以及Leader的最后一个日志条目的索引，并进行排序，取中位数作为新的提交进度，因为这个索引位置的日志条目能够在集群超过半数节点上安全提交。



```java
if (raftLog.getEntryTerm(newCommitIndex) != currentTerm) {
    return;
}
if (commitIndex >= newCommitIndex) {
    return;
}
```

> 如果新的提交进度索引位置的日志条目的任期号不是当前任期号，受安全性限制，不能推进提交进度，直接返回。
>
> 如果新的提交进度小于等于CommitIndex，则无需推进提交进度，直接返回。



```java
long oldCommitIndex = commitIndex;
commitIndex = newCommitIndex;
raftLog.updateMetaData(currentTerm, null, raftLog.getFirstLogIndex(), commitIndex);
for (long index = oldCommitIndex + 1; index <= newCommitIndex; index++) {
    RaftProto.LogEntry entry = raftLog.getEntry(index);
    if (entry.getType() == RaftProto.EntryType.ENTRY_TYPE_DATA) {
        stateMachine.apply(entry.getData().toByteArray());
    } else if (entry.getType() == RaftProto.EntryType.ENTRY_TYPE_CONFIGURATION) {
        applyConfiguration(entry);
    }
}
lastAppliedIndex = commitIndex;
commitIndexCondition.signalAll();
```

> 设置CommitIndex为新的提交进度。
>
> 遍历旧的提交进度到新的提交进度之间的日志条目。
>
> 如果是一般日志条目，调用**StateMachine**接口的**apply()**方法，应用到当前节点的复制状态机。
>
> 如果是集群配置日志条目，则调用**applyConfiguration()**方法，更新当前节点维护的集群配置信息。
>
> 将已应用到复制状态机的最后一个日志条目索引更新为CommitIndex，表明所有已提交的日志条目都被应用到复制状态机。
>
> 唤醒在同步锁的等待条件中等待的所有客户端线程。



##### 日志复制：更新集群配置信息的applyConfiguration()方法

```java
try {
    RaftProto.Configuration newConfiguration
            = RaftProto.Configuration.parseFrom(entry.getData().toByteArray());
    configuration = newConfiguration;
    for (RaftProto.Server server : newConfiguration.getServersList()) {
        if (!peerMap.containsKey(server.getServerId())
                && server.getServerId() != localServer.getServerId()) {
            Peer peer = new Peer(server);
            peer.setNextIndex(raftLog.getLastLogIndex() + 1);
            peerMap.put(server.getServerId(), peer);
        }
    }
} catch (InvalidProtocolBufferException ex) {
    ex.printStackTrace();
}
```

> 从集群配置日志条目中解析新的集群配置信息。
>
> 遍历新的集群配置信息中的每个节点，如果不是当前节点，且还未加入节点状态信息集合中，则加入到集合中，同时为每个其他节点维护NextIndex。



#### 日志存储实现SegmentedLog类

##### 成员属性

```java
private String logDir;
private String logDataDir;
private int maxSegmentFileSize;
private RaftProto.LogMetaData metaData;
private TreeMap<Long, Segment> startLogIndexSegmentMap = new TreeMap<>();
private volatile long totalSize;
```

> 维护了日志目录、日志数据目录、单个日志文件的最大大小、日志文件的元信息、日志文件信息集合（按照第一条日志条目的索引排序）、日志文件的总大小。



##### 构造器

```java
this.logDir = raftDataDir + File.separator + "log";
this.logDataDir = logDir + File.separator + "data";
this.maxSegmentFileSize = maxSegmentFileSize;
```

> 初始化日志目录、日志数据目录、单个日志文件的最大大小。



```java
File file = new File(logDataDir);
if (!file.exists()) {
    file.mkdirs();
}
readSegments();
for (Segment segment : startLogIndexSegmentMap.values()) {
    this.loadSegmentData(segment);
}
```

> 创建日志数据目录路径的文件夹。
>
> 调用**readSegments()**方法，加载日志文件信息到日志文件信息集合。
>
> 遍历日志文件信息集合中的每个日志文件，调用**loadSegmentData()**方法，加载日志文件数据。



```java
metaData = this.readMetaData();
if (metaData == null) {
    if (startLogIndexSegmentMap.size() > 0) {
        throw new RuntimeException("No readable metadata file but found segments");
    }
    metaData = RaftProto.LogMetaData.newBuilder().setFirstLogIndex(1).build();
}
```

> 初始化元信息。



##### 加载日志文件信息的readSegments()方法

```java
try {
    List<String> fileNames = RaftFileUtils.getSortedFilesInDirectory(logDataDir, logDataDir);
    for (String fileName : fileNames) {
        ...
    }
} catch(IOException ioException){
    throw new RuntimeException("open segment file error");
}
```

> 获取日志数据目录下的日志文件列表，并进行排序。遍历日志文件列表，加载日志文件信息。



```java
for (String fileName : fileNames) {
    String[] splitArray = fileName.split("-");
    if (splitArray.length != 2) {
        continue;
    }

    ...
}
```

> 提取日志文件名称中的信息，日志文件名称由两部分构成，中间使用"-"号连接，不符合此命名格式的文件将被跳过。



```java
for (String fileName : fileNames) {
    ...

    Segment segment = new Segment();
    segment.setFileName(fileName);
    if (splitArray[0].equals("open")) {
        segment.setCanWrite(true);
        segment.setStartIndex(Long.valueOf(splitArray[1]));
        segment.setEndIndex(0);
    } else {
        ...
    }
    
    ...
}
```

> 如果日志文件名称的第一部分为"open"，表示此日志文件是最新的日志文件，日志文件名称的第二部分表示日志文件的第一个日志条目的索引。设置此日志文件为可写。



```java
for (String fileName : fileNames) {
    ...

    if (splitArray[0].equals("open")) {
        ...
    } else {
        try {
            segment.setCanWrite(false);
            segment.setStartIndex(Long.parseLong(splitArray[0]));
            segment.setEndIndex(Long.parseLong(splitArray[1]));
        } catch (NumberFormatException ex) {
            continue;
        }
    }

    ...
}
```

> 如果日志文件名称的第一部分不为"open"，表示此日志文件是已经写满的日志文件，日志文件名称的第一部分表示日志文件的第一个日志条目的索引，第二部分表示日志文件的最后一个日志条目的索引。设置此日志文件为不可写。
>
> 如果此文件名称不能按上述格式解析，则跳过此文件。



```java
for (String fileName : fileNames) {
    ...

    segment.setRandomAccessFile(RaftFileUtils.openFile(logDataDir, fileName, "rw"));
    segment.setFileSize(segment.getRandomAccessFile().length());
    startLogIndexSegmentMap.put(segment.getStartIndex(), segment);
}
```

> 获取日志文件及文件大小。
>
> 将日志文件信息放入日志文件信息集合，并按日志文件的第一个日志条目的索引排序。



##### 加载日志文件数据的loadSegmentData()方法

```java
try {
    RandomAccessFile randomAccessFile = segment.getRandomAccessFile();
    long totalLength = segment.getFileSize();
    long offset = 0;
    while (offset < totalLength) {
        RaftProto.LogEntry entry = RaftFileUtils.readProtoFromFile(
                randomAccessFile, RaftProto.LogEntry.class);
        if (entry == null) {
            throw new RuntimeException("read segment log failed");
        }
        Segment.Record record = new Segment.Record(offset, entry);
        segment.getEntries().add(record);
        offset = randomAccessFile.getFilePointer();
    }
    totalSize += totalLength;
} catch (Exception ex) {
    throw new RuntimeException("file not found");
}
```

> 打开日志文件，逐个加载日志条目。
>
> 如果加载的日志条目为空，抛出异常，否则加入日志条目集合，读取下一个日志条目。
>
> 将此日志文件中的日志条目全部加载完成后，累加日志文件的总大小。



```java
int entrySize = segment.getEntries().size();
if (entrySize > 0) {
    segment.setStartIndex(segment.getEntries().get(0).entry.getIndex());
    segment.setEndIndex(segment.getEntries().get(entrySize - 1).entry.getIndex());
}
```

> 如果从日志文件中加载到日志条目，则更新日志文件的第一个日志条目的索引和最后一个日志条目的索引。



##### 追加日志条目到日志文件的append()方法

```java
long newLastLogIndex = this.getLastLogIndex();
for (RaftProto.LogEntry entry : entries) {
    newLastLogIndex++;
    int entrySize = entry.getSerializedSize();
    int segmentSize = startLogIndexSegmentMap.size();
    boolean isNeedNewSegmentFile = false;
    try {
        ...
    }  catch (IOException ex) {
        throw new RuntimeException("append raft log exception, msg=" + ex.getMessage());
    }
}
return newLastLogIndex;
```

> 获取当前节点的最后一个日志条目的索引，从该索引位置开始，将日志条目逐个追加到日志文件中，更新最后一个日志条目的索引。
>
> 追加完成后，返回最后一个日志条目的索引。



```java
try {
    if (segmentSize == 0) {
        isNeedNewSegmentFile = true;
    } else {
        Segment segment = startLogIndexSegmentMap.lastEntry().getValue();
        if (!segment.isCanWrite()) {
            isNeedNewSegmentFile = true;
        } else if (segment.getFileSize() + entrySize >= maxSegmentFileSize) {
            isNeedNewSegmentFile = true;
            segment.getRandomAccessFile().close();
            segment.setCanWrite(false);
            String newFileName = String.format("%020d-%020d",
                                               segment.getStartIndex(), segment.getEndIndex());
            String newFullFileName = logDataDir + File.separator + newFileName;
            File newFile = new File(newFullFileName);
            String oldFullFileName = logDataDir + File.separator + segment.getFileName();
            File oldFile = new File(oldFullFileName);
            FileUtils.moveFile(oldFile, newFile);
            segment.setFileName(newFileName);
            segment.setRandomAccessFile(RaftFileUtils.openFile(logDataDir, newFileName, "r"));
        }
    }

	...
}  catch (IOException ex) {
	...
}
```

> 满足以下任意一个条件时，需要创建新的日志文件：
>
> 1. 不存在已创建的日志文件。
> 2. 最后一个日志文件为不可写。
> 3. 最后一个日志文件为可写，但剩余空间不足以写入当前待追加的日志条目。
>
> 如果满足上述第三个条件，则还需要将该日志文件进行结束处理。
>
> 关闭该日志文件，设置该日志文件为不可写。
>
> 根据该日志文件的第一个日志条目的索引和最后一个日志条目的索引，创建固定格式名称的新日志文件，并将该日志文件的数据复制到新日志文件中，并删除该日志文件。
>
> 更新日志文件信息。
>
> 重新打开日志文件。



```java
try {
	...

    Segment newSegment;
    if (isNeedNewSegmentFile) {
        String newSegmentFileName = String.format("open-%d", newLastLogIndex);
        String newFullFileName = logDataDir + File.separator + newSegmentFileName;
        File newSegmentFile = new File(newFullFileName);
        if (!newSegmentFile.exists()) {
            newSegmentFile.createNewFile();
        }
        Segment segment = new Segment();
        segment.setCanWrite(true);
        segment.setStartIndex(newLastLogIndex);
        segment.setEndIndex(0);
        segment.setFileName(newSegmentFileName);
        segment.setRandomAccessFile(RaftFileUtils.openFile(logDataDir, newSegmentFileName, "rw"));
        newSegment = segment;
    } else {
        newSegment = startLogIndexSegmentMap.lastEntry().getValue();
    }

    ...
}  catch (IOException ex) {
	...
}
```

> 如果需要创建新的日志文件，则创建固定格式名称的新日志文件，设置新日志文件为可写，并更新日志文件信息。
>
> 如果不需要创建新的日志文件，则获取最后一个日志文件。



```java
try {
	...

    if (entry.getIndex() == 0) {
        entry = RaftProto.LogEntry.newBuilder(entry)
                .setIndex(newLastLogIndex).build();
    }
    newSegment.setEndIndex(entry.getIndex());
    newSegment.getEntries().add(new Segment.Record(
            newSegment.getRandomAccessFile().getFilePointer(), entry));
    RaftFileUtils.writeProtoToFile(newSegment.getRandomAccessFile(), entry);
    newSegment.setFileSize(newSegment.getRandomAccessFile().length());
    if (!startLogIndexSegmentMap.containsKey(newSegment.getStartIndex())) {
        startLogIndexSegmentMap.put(newSegment.getStartIndex(), newSegment);
    }
    totalSize += entrySize;
}  catch (IOException ex) {
	...
}
```

> 如果当前待追加的日志条目没有设置索引，则当前节点是Leader节点，将该日志条目的索引设置为最后一个日志条目的下一个索引。
>
> 更新日志文件信息。
>
> 将日志条目写入日志文件。
>
> 如果日志文件信息集合中没有此日志文件的信息，即此日志文件是新的日志文件，则将新日志文件信息加入到日志文件信息集合。
>
> 累加日志文件的总大小。



##### 删除头部所有日志条目的truncatePrefix()方法

```java
if (newFirstIndex <= getFirstLogIndex()) {
    return;
}
```

> 如果需要保留的第一个日志条目的索引小于等于当前节点的第一个日志条目的索引，则无需删除日志条目，直接返回。



```java
long oldFirstIndex = getFirstLogIndex();
while (!startLogIndexSegmentMap.isEmpty()) {
    Segment segment = startLogIndexSegmentMap.firstEntry().getValue();
    if (segment.isCanWrite()) {
        break;
    }
    if (newFirstIndex > segment.getEndIndex()) {
        File oldFile = new File(logDataDir + File.separator + segment.getFileName());
        try {
            RaftFileUtils.closeFile(segment.getRandomAccessFile());
            FileUtils.forceDelete(oldFile);
            totalSize -= segment.getFileSize();
            startLogIndexSegmentMap.remove(segment.getStartIndex());
        } catch (Exception ex2) {
            ...
        }
    } else {
        break;
    }
}
```

> 进行循环，按顺序遍历日志文件信息集合中的日志文件并进行删除，直到满足以下任意一个条件：
>
> 1. 第一个日志文件为可写。
> 2. 第一个日志文件的最后一个日志条目的索引大于等于需要保留的第一个日志条目的索引。
> 3. 所有日志文件都被删除。
>
> 如果日志文件的最后一个日志条目的索引小于需要保留的第一个日志条目的索引，则该日志文件需要删除。
>
> 关闭该日志文件，强制删除该日志文件。
>
> 累减日志文件的总大小。
>
> 将该日志文件信息从日志文件信息集合中移除。



```java
long newActualFirstIndex;
if (startLogIndexSegmentMap.size() == 0) {
    newActualFirstIndex = newFirstIndex;
} else {
    newActualFirstIndex = startLogIndexSegmentMap.firstKey();
}
updateMetaData(null, null, newActualFirstIndex, null);
```

> 如果全部日志文件被删除，则当前节点的第一个日志条目的索引为需要保留的第一个日志条目的索引。
>
> 如果保留了日志文件，则当前节点的第一个日志条目的索引为第一个日志文件的第一个日志条目的索引。
>
> 更新元信息。



##### 删除尾部所有日志条目的truncateSuffix()方法

```java
if (newEndIndex >= getLastLogIndex()) {
    return;
}
```

> 如果需要保留的最后一个日志条目的索引大于等于当前节点的最后一个日志条目的索引，则无需删除日志条目，直接返回。



```java
while (!startLogIndexSegmentMap.isEmpty()) {
    Segment segment = startLogIndexSegmentMap.lastEntry().getValue();
    try {
        if (newEndIndex == segment.getEndIndex()) {
            break;
        } else if (newEndIndex < segment.getStartIndex()) {
            totalSize -= segment.getFileSize();
            segment.getRandomAccessFile().close();
            String fullFileName = logDataDir + File.separator + segment.getFileName();
            FileUtils.forceDelete(new File(fullFileName));
            startLogIndexSegmentMap.remove(segment.getStartIndex());
        } else if (newEndIndex < segment.getEndIndex()) {
            ...
        }
    } catch (IOException ex) {
        ...
    }
}
```

> 进行循环，按逆序遍历日志文件信息集合中的日志文件并进行删除，直到满足以下任意一个条件：
>
> 1. 最后一个日志文件的最后一个日志条目的索引等于需要保留的最后一个日志条目的索引。
> 2. 所有日志文件都被删除。
>
> 如果日志文件的第一个日志条目的索引大于需要保留的最后一个日志条目的索引，则该日志文件需要删除。
>
> 关闭该日志文件，强制删除该日志文件。
>
> 累减日志文件的总大小。
>
> 将该日志文件信息从日志文件信息集合中移除。



```java
if (newEndIndex == segment.getEndIndex()) {
    ...
} else if (newEndIndex < segment.getStartIndex()) {
    ...
} else if (newEndIndex < segment.getEndIndex()) {
    int i = (int) (newEndIndex + 1 - segment.getStartIndex());
    segment.setEndIndex(newEndIndex);
    long newFileSize = segment.getEntries().get(i).offset;
    totalSize -= (segment.getFileSize() - newFileSize);
    segment.setFileSize(newFileSize);
    segment.getEntries().removeAll(
            segment.getEntries().subList(i, segment.getEntries().size()));
    FileChannel fileChannel = segment.getRandomAccessFile().getChannel();
    fileChannel.truncate(segment.getFileSize());
    fileChannel.close();
    segment.getRandomAccessFile().close();
    String oldFullFileName = logDataDir + File.separator + segment.getFileName();
    String newFileName = String.format("%020d-%020d",
            segment.getStartIndex(), segment.getEndIndex());
    segment.setFileName(newFileName);
    String newFullFileName = logDataDir + File.separator + segment.getFileName();
    new File(oldFullFileName).renameTo(new File(newFullFileName));
    segment.setRandomAccessFile(RaftFileUtils.openFile(logDataDir, segment.getFileName(), "rw"));
}
```

> 如果需要保留的最后一个日志条目的索引大于等于日志文件的第一个日志条目的索引，且小于日志文件的最后一个日志条目的索引，则该日志文件需要删除部分日志条目数据。
>
> 累减日志文件的总大小。
>
> 获取需要删除的第一个日志条目在日志文件中的偏移量，删除日志文件该偏移量之后的数据。
>
> 关闭该日志文件。
>
> 根据该日志文件的第一个日志条目的索引和最后一个日志条目的索引，修改日志文件名称。
>
> 更新日志文件信息。
>
> 重新打开日志文件。
