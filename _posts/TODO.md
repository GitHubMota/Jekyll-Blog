问题现象

21个主节点21个从节点集群进行数据扩容,添加15个主节点15个从节点.
添加过程中会遍历集群各节点执行cluster info命令检查cluster_known_nodes是否达到21*2+15*2=72. 如果超过则报错进行回滚操作(回滚时执行cluster forget新加入的节点并进行新节点下线操作).
结果扩容失败，监控日志中出现该异常:
Redis<10.240.0.153:5067 db:0>:fatal error, found [74] nodes more than we expected [72]

后面再次执行扩容操作，初始检查到节点数为43，导致43+15*2=73> 72，继续报异常：
[error] fatal error, found [73] nodes more than we expected [72] 

查看该集群节点信息,发现有节点A(10.240.50.233:5055)和B的状态是fail和handshake. 而且不同节点上A和B的nodeid不一样，状态也有的是handshake，有的是fail.

分析

查看机器监控节点A的日志，在扩容时添加节点A并启动，扩容失败时销毁. 现在的状态应该是已经下线了的. 但是看它在某节点C显示的状态为handshake，隔一会就改变一次nodeid. 通过cluster forget清除节点A信息,清除后隔一会又会出现状态为handshake且改变了nodeid的节点A信息.

这种情况看来cluster forget就无法恢复集群状态了，问题的关键还是找到触发nodeid变化的原因.


怀疑节点A销毁失败，搜索节点A进程不存在.

怀疑节点A netstat 查询节点A redis端口及其cluster bus端口也不存在.

Config set loglevel debug打开节点C的调试日志.
观察到节点C一直在尝试连接节点A的cluster bus端口然后失败.
158542:S 04 Jun 15:31:14.820 . Connecting with Node 0bcbe14df0fb83b9f4190aa115bb522121b18a5b at 10.240.50.233:15055
158542:S 04 Jun 15:31:14.820 . I/O error reading from node link: Connection refused


查看nodeid发生变化中间的日志
GOSSIP 0bcbe14df0fb83b9f4190aa115bb522121b18a5b 10.240.50.233:5055 master,fail

可知为收到包含节点A fail的gossip信息后节点C记录的节点A nodeid发生变化.

搜索代码中打印"GOSSIP "定位到clusterProcessGossipSection()

void clusterProcessGossipSection(clusterMsg *hdr, clusterLink *link) {
    uint16_t count = ntohs(hdr->count);
    clusterMsgDataGossip *g = (clusterMsgDataGossip*) hdr->data.ping.gossip;
    clusterNode *sender = link->node ? link->node : clusterLookupNode(hdr->sender);

    while(count--) {
        uint16_t flags = ntohs(g->flags);
        clusterNode *node;
        sds ci;

        ci = representClusterNodeFlags(sdsempty(), flags);
        serverLog(LL_DEBUG,"GOSSIP %.40s %s:%d %s",
            g->nodename,
            g->ip,
            ntohs(g->port),
            ci);
        sdsfree(ci);

        /* Update our state accordingly to the gossip sections */
        node = clusterLookupNode(g->nodename);//节点C中记录的节点A nodeid一直在变化，与goosip报文中的不一样，所以返回node为null
        if (node) {
……
        } else {
            /* If it's not in NOADDR state and we don't have it, we
             * start a handshake process against this IP/PORT pairs.
             *
             * Note that we require that the sender of this gossip message
             * is a well known node in our cluster, otherwise we risk
             * joining another cluster. */
            if (sender &&
                !(flags & CLUSTER_NODE_NOADDR) &&
                !clusterBlacklistExists(g->nodename))
            {
                clusterStartHandshake(g->ip,ntohs(g->port));
            }
        }

        /* Next node */
        g++;
    }
}

搜索代码中改变nodeid(node->name)定位到createClusterNode()，进入到clusterStartHandshake()->createClusterNode()->getRandomHexChars(node->name, CLUSTER_NAMELEN)随机生成nodeid，进入handshake状态.  (如果下线的节点重新上线了，与该节点成功建立连接，并在收到该节点报文后更新其nodeid为节点真正的nodeid)

nodeid变化的原因找到，且handshake状态的nodeid会发生变化，fail状态的不会变化，查看集群状态信息后能印证该观点.

解决方法
既然handshake状态是由于收到fail状态信息导致的，那么只用把fail状态forget掉就可以，而且fail状态的节点A nodeid是一直不变的.

在集群的每个节点，执行cluster forget其节点包含fail状态节点的nodeid, 之后handshake状态信息也不见了，再次执行数据扩容操作，成功完成.

如何复现这种集群状态呢.

运维说之前手动执行过cluster forget操作. 找了个测试环境3主3从的集群, 选择其中一个主节点A，先forget从节点B，然后下线B，随后节点A出现B的handshake状态，其他节点显示B为fail状态. (注意如果是下线主节点，3主集群会由于达不到大多数而无法判定节点到fail状态)

通过redis-cli去循环获取节点A的cluster nodes信息，可以看到每隔15秒B的nodeid会发生变化.15秒刚好是cluster-node-timeout的配置，查看代码中clusterCron():
void clusterCron(void) {
……
handshake_timeout = server.cluster_node_timeout;
…...
        if (nodeInHandshake(node) && now - node->ctime > handshake_timeout) {
            clusterDelNode(node);
            continue;
        }
…...

即当handshake持续超过节点配置的超时时间，则从该节点的nodes里删除该handshake节点.然后下次收到goosip带该节点fail的信息又开始handshake.. 并且如果在handshake期间收到goosip消息，由于handshake nodeid一直变化，依然会进入到clusterStartHandshake(),但是该函数里面执行了clusterHandshakeInProgress()判断以防止相同ip:port多次handshake.

去网上搜索到Cluster: How to remove a node in handshake state[https://github.com/antirez/redis/issues/2965]

提问者提出:
Cluster scale: 512 nodes, one master have three salves connected.
In the cluster one node stay in handshake state and the node has been failed down, so in cluster nodes we can see the node id changed but can not join to cluster.
How to remove this handshake node from the cluster?

作者指出原因:
There are only two ways this can happen:
	1. You fail to send CLUSTER FORGET to all the nodes in the cluster. So eventually there are nodes that still has a clue about this other node, and it will inform the other nodes via gossip. Make sure to send CLUSTER FORGET to every single node in the cluster.
	2. Or alternatively, there is an instance running in 10.15.107.150 but you said there is not.

提问者均保证第二点是没问题的，最终确认原因就是下线节点前执行的cluster forget在某些节点由于诸如网络不稳定的原因导致失败。所以只要那些失败的节点继续执行cluster forget即可，而那些标志下线节点为fail状态的就是前面执行cluster forget失败的.

最后有人提供了个脚本执行cluster forget:
#echo "usage: host port"
nodes_addrs=$(redis-cli -h $1 -p $2 cluster nodes|grep -v handshake| awk '{print $2}')
echo $nodes_addrs
for addr in ${nodes_addrs[@]}; do
    host=${addr%:*}
    port=${addr#*:}
    del_nodeids=$(redis-cli -h $host -p $port cluster nodes|grep -E 'handshake|fail'| awk '{print $1}')
    for nodeid in ${del_nodeids[@]}; do
        echo $host $port $nodeid
        redis-cli -h $host -p $port cluster forget $nodeid
    done
done

虽然我认为只用forget fail状态的，handshake状态会在超时后自动删除，不过这个脚本解决这个问题还是没什么毛病...话说以后有问题还是先来github搜搜吧.

