# Zookeeper

入口应用`QuorumPeerMain`

节点存储使用ConcurrentHashMap

接收命令时

- 持久化命令（记录日志、快照，将DataTree序列化保存到文件中就是生成快照）
- 修改DataTree

ZookeeperServer启动时：

1. 解析配置文件
2. 开启定时器去定时删除日志和快照，默认保存三个 
3. 根据快照构造DataTree
4. 创建Socket监听端口

根据配置文件中的信息初始化QuorumPeerConfig