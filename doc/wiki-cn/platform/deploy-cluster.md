# 集群部署

本小节主要介绍创建一套集群的全部流程，包括
1. 资源获取
2. 资源分布
3. 主从分布
3. 集群一致性检验  

以及[如何快速创建一套redis cluster](#一种利用nodes.conf快速创建集群的办法)
## 资源获取
当framework收到集群创建任务的时候，首先要向mesos请求恢复资源(call.Revive），确保mesos集群有足够的资源能够部署当前任务。收到revice后，mesos会向framework回复当前平台可用的资源offer，scheduler根据当前offer判断是否满足集群部署的需求。

## 资源分布
当获取到的offer有足够的资源可以部署当前创建任务时，scheduler根据算法规则[chunk/dist](chunk.md)进行集群节点分布规划。节点分布规划包括两部分 每个主机上应该部署几个节点；cluster的主从要如何分布。规划节点分布的同时，scheduler会生成具体的节点创建计划，并下发到executor创建具体的节点实例。

## 主从分布
对于redis cluster，为了保证集群的高可用，必须确保集群的主从不能同时分布在一台host上，为了解决这一问题，我们使用了[chunk算法](chunk.md)对节点进行分组。通过（A主B从）-（A从B主）这样一对互为主从的chunk组，保证了主从分布的高可用。

## 集群一致性校验
redis cluster启动后，为了保证集群的真实可用，以及集群的分布的确服务预先创建的chunk规则，我们还需要对进群进行一致性校验，确保所有节点已经通过gossip达成了最终一致，并且cluster status处于ok状态。

## 一种利用nodes.conf快速创建集群的办法
传统的redis cluster创建方式是各个节点以cluster的模式启动，节点启动完毕后，通过redis-trib.rb工具进行节点间的握手。redis-trib通过向每一个节点发送CLUSTER MEET请求，来进行节点间的握手，节点握手成功后，通过Gossip协议进行集群间nodes节点信息的同步。但是，单节点数增长到一定程度的时候，通过CLUSTER MEET  进行集群握手将会有巨大的效率问题，由于O(n^2)的复杂度，当节点数量进一步增长时，握手效率也会急速下降。

为了解决这一问题，我们模拟了集群恢复的流程，通过mock nodes.conf的方式提前创建好nodes文件。通过mock nodes.conf 同样是创建600个主节点，使用新方法仅仅需要10s的时间，速度提升了百倍不止。

以下是一个mock nodes.conf 的例子

```
0000000000000000000000000000000000225157 172.22.33.199:31000@41000 master - 0 0 0 connected 0-4096
0000000000000000000000000000000000225158 172.22.33.199:31001@41001 myself,slave 0000000000000000000000000000000000225159 0 0 0 connected
0000000000000000000000000000000000225159 172.22.33.184:31000@41000 master - 0 0 0 connected 4097-8193
0000000000000000000000000000000000225160 172.22.33.184:31001@41001 slave 0000000000000000000000000000000000225157 0 0 0 connected
0000000000000000000000000000000000225161 172.22.33.192:31000@41000 master - 0 0 0 connected 8194-12290
0000000000000000000000000000000000225162 172.22.33.192:31001@41001 slave 0000000000000000000000000000000000225163 0 0 0 connected
0000000000000000000000000000000000225163 172.22.33.187:31000@41000 master - 0 0 0 connected 12291-16383
0000000000000000000000000000000000225164 172.22.33.187:31001@41001 slave 0000000000000000000000000000000000225161 0 0 0 connected
vars currentEpoch 0 lastVoteEpoch 0
```

通过提前为每一个节点生成runnid，并渲染nodes.conf模板，节点启动时，只需要根据nodes.conf进行集群的恢复，而无需进行漫长的meet握手，大大提升了集群创建的速度。