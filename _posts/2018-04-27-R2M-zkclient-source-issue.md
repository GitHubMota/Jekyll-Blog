---
layout: post
title:  R2M客户端zk连接未关闭问题定位
date:   2018-04-27 16:01:23 +0800
categories: Dev
tag: zookeeper
---

* content
{:toc}

问题现象
----------

某用户在使用R2M系统过程中,web上监控到其连接到集群上客户端使用数量一直在增加. 运维与其沟通后未能定位问题.

听用户对问题的描述及复现, 用户启动某进程,然后进程中启动线程执行任务,任务开始时新建客户端,执行大量集群读写命令,任务终止时则调用客户端的close().期望是每个任务结束后客户端关闭,但结果是客户端不能关闭,web上监控其客户端数量只有增加,没有减少.

此时客户已经在其开发环境中复现问题,并一次任务启动多个客户端,再去终止任务,通过日志查看确实已调用与启动客户端等量的close().但web依然监控到客户端增加.


问题确认
----------
与其确认使用的`r2m_client`是什么版本,若版本低于1.4.8则建议升级,简单的客户端使用步骤常见问题排除...

查看该显示的客户端信息是什么.阅读web展示的代码,其客户端信息是从zookeeper某clients/path中获得的,直接连接到zk上看到的与其显示的一致. 

连接到redis集群上client list查看,所有与redis集群中各节点接收读写命令的连接在任务启动时会建立,并且任务终止时正常关闭.该连接为用户核心功能.

源码及日志分析
----------
新建客户端代码逻辑:
指定zk地址及集群名称,客户端通过zk获得集群中节点信息,对应每个节点创建一个连接池,之后客户端命令通过key hash值对应到某节点的连接池上,从连接池中获得与redis节点的连接使用.  

然后是启动线程新建zk会话,监听一个command/path,当集群节点/配置等信息发生变化时重新加载. 注册该客户端到clients/path(创建一个临时节点(EPHEMERAL),临时节点的生命周期和客户端会话绑定.也就是说,如果客户端会话失效,那么这个节点就会自动被清除掉.)

查看客户端的close(),里面依次执行了关闭zk会话,关闭redis各节点连接.

从日志中看客户端启动和终止是成对出现的,同时redis连接也正常关闭,说明每个客户端调用了close().

关闭zk会话的代码
    
    ZkClient::close()
        public void close() throws ZkInterruptedException {
            if (!this._closed) {
                LOG.debug("Closing ZkClient...");
                this.getEventLock().lock();
    
                try {
                    this.setShutdownTrigger(true);
                    this._eventThread.interrupt();
                    this._eventThread.join(2000L);
                    this._connection.close();
                    this._closed = true;
                } catch (InterruptedException var5) {
                    throw new ZkInterruptedException(var5);
                } finally {
                    this.getEventLock().unlock();
                }
    
                LOG.debug("Closing ZkClient...done");
            }
        }

Zkclient close执行前打印`Closing ZkClient...`,执行后`Closing ZkClient...done`.并且在上一级调用处会打印`[Notify Service] close ZkNotifyService`.

建议用户打开日志debug级别,并减少并发(方便查看)后复现,查看该流程是否正常. 

该次日志中发现所有注册zkclient的日志均出现在`Closing ZkClient...done`后面. 看起来是close()在注册zkclient之前调用,与用户沟通该次运行new client和close中间没有执行命令操作,即new完client后立即close,考虑到注册zkclient是异步线程调用的,思考是否因为close()执行太快,异步线程在其结束后调用注册zkclient导致连接未关闭. 

建议用户在close()前加入sleep等待1s. 结果再次运行后连接正常关闭. 

向用户指出该问题原因为异步的zk连接完成前调用close()导致,用户提出zk连接尚未完成为什么与redis节点的连接是正常的. 

查看代码进行确认：客户端与redis连接的处理与该注册zkclient是分开的,前者也有一次与zk建立连接获取节点信息已建立redis节点连接然后关闭zk连接的操作. 该过程也解释了`Closing ZkClient...`日志比用户调用close()数量多的问题.

但是用户提出在线上是在运行好几天后停止任务zk连接未关闭的,此时zk连接应该早建立完成了.

在此期间我在本机上并发去模拟该情况时也并未出现该问题,对该原因也抱有怀疑,查看close()也发现其有去结束initZkClient的线程,不应该在close完后还有注册zkclient的情况.

`r2m_client` Close()代码:
    
    if (!initWorkingThread.isInterrupted()) {
        try {
            initWorkingThread.interrupt();
        } catch (Exception e) {
            //          e.printStackTrace();
        }
    }

于是建议用户按线上情况模拟,即在new client后等待一段时间再去结束任务以调用close().于是复现zk连接未关闭的问题. 之后再运行一版close()前sleep的日志,该情况下zk连接能正常关闭.

查看日志,两者zkclient均建立并注册完成的,但是发现未加sleep的日志中`Closing ZkClient...done`的数量比`Closing ZkClient…`少,并且少的数量刚好是未关闭zk连接的数量,即存在进行zk连接关闭操作但是失败的情况.

至此问题定位到这两行日志中间哪个步骤出错, 首先怀疑是线程间死锁问题,建议用户将thread信息打印出来,随后观察到每次注册zkclient的线程只有一个是与其他不一样的,而刚好这个线程的zkclient能正常关闭. 

询问用户控制并发的代码逻辑,用户表示为flink调度管理的,也不知道为什么这个线程与别的不一样.

查看zkclient.close()中有捕获并抛出异常ZkInterruptedException.但是在`r2m_client`调用zkclient.close()处却捕获并忽略该异常了.

    if (serviceClient != null) {
        try {
            serviceClient.unsubscribeAll();
            serviceClient.close();
        } catch (Exception e) {
            //          exceptionHandle.log(e);
        } finally {
            serviceClient = null;
        }
    }

将该注释打开并添加日志,重新打包一版`r2m_client`发送给用户.用户运行后确实出现异常：

        org.I0Itec.zkclient.exception.ZkInterruptedException: java.lang.InterruptedException
        at org.I0Itec.zkclient.ZkClient.close(ZkClient.java:1268)
        at com.wangyin.rediscluster.notification.service.ZkNotifyService.close(ZkNotifyService.java:249)
        at com.wangyin.rediscluster.client.OriginalCacheClusterClient.close(OriginalCacheClusterClient.java:167)
        at com.jd.jr.smart.data.flink.r2m.R2MSink.close(R2MSink.java:131)
        at org.apache.flink.api.common.functions.util.FunctionUtils.closeFunction(FunctionUtils.java:43)
        at org.apache.flink.streaming.api.operators.AbstractUdfStreamOperator.dispose(AbstractUdfStreamOperator.java:117)
        at org.apache.flink.streaming.runtime.tasks.StreamTask.disposeAllOperators(StreamTask.java:446)
        at org.apache.flink.streaming.runtime.tasks.StreamTask.invoke(StreamTask.java:351)
        at org.apache.flink.runtime.taskmanager.Task.run(Task.java:718)
    at java.lang.Thread.run(Unknown Source)
        Caused by: java.lang.InterruptedException
        at java.lang.Object.wait(Native Method)
        at java.lang.Thread.join(Unknown Source)
    at org.I0Itec.zkclient.ZkClient.close(ZkClient.java:1264)
        ... 9 more

异常定位
----------
看到zkclient代码的异常表示很尴尬,查看github zkclient代码仓库中看该处没有更新.

在网上google到[Helix JIRA issue](https://issues.apache.org/jira/browse/HELIX-264)记录了该问题,

[helix](https://blog.csdn.net/oopsoom/article/details/47416575)和用户的[flink](https://www.jianshu.com/p/2ee7134d7373)都会有分布式并行调度的操作, 故该问题在flink使用zkclient上应该也是会同样复现的.

Helix JIRA中提到该原因:

    This is probably a zkclient bug that we should never call zkclient.close() from its own event thread context. 

意思是调用 zkclient.close() 的这个线程不能是_eventThread.join()的这个_eventThread.

从日志中查看确实出现的这种情况.思考用户场景如下:

先是终止任务,则该任务中的线程包括event thread收到中断,如果添加sleep,则close()在event thread中断完成后被调用,此时正常,若不添加,则close()调用join时event thread尚在运行,抛出异常.

join/wait 异常堆栈代码

        public final synchronized void join(long millis)
        throws InterruptedException {
            …
                while (isAlive()) { //thread线程未停止则进入
                    long delay = millis - now;
                    if (delay <= 0) {
                        break;
                    }
                    wait(delay);
                    now = System.currentTimeMillis() - base;
                }
        }
    }
    
    /*...
     * @throws  InterruptedException if any thread interrupted the
     *             current thread before or while the current thread
     *             was waiting for a notification.  The <i>interrupted
     *             status</i> of the current thread is cleared when
     *             this exception is thrown.
     * @see        java.lang.Object#notify()
     * @see        java.lang.Object#notifyAll()
     */
    public final native void wait(long timeout) throws InterruptedException;

看wait()说明,由于event thread未停止,调用wait,而该方法等待过程中如果本线程(同样是event thread)被中断则抛出InterruptedException 异常.

至此从源码中也能印证添加sleep与不添加sleep产生的影响,已找到原因.

至于为什么flink调度会出现调用zkclient.close()的这个线程是_eventThread.join()的这个_eventThread的场景不得而知.

解决方案
----------
在网上搜索到某[解决方法](https://www.programcreek.com/java-api-examples/?api=org.I0Itec.zkclient.exception.ZkInterruptedException)如下,其对zkclient.close()做出了修改,捕获异常并对当前线程的中断状态暂存并关闭zk连接.

    /**
     * Close the client.
     *
     * @throws ZkInterruptedException
     */
    public void close() throws ZkInterruptedException {
        if (LOG.isTraceEnabled()) {
            StackTraceElement[] calls = Thread.currentThread().getStackTrace();
            LOG.trace("closing a zkclient. callStack: " + Arrays.asList(calls));
        }
        getEventLock().lock();
        try {
            if (_connection == null || _closed) {
                return;
            }
            LOG.info("Closing zkclient: " + ((ZkConnection) _connection).getZookeeper());
            setShutdownTrigger(true);
            _eventThread.interrupt();
            _eventThread.join(2000);
            _connection.close();
            _closed = true;
        } catch (InterruptedException e) {
            /**
             * Workaround for HELIX-264: calling ZkClient#close() in its own eventThread context will
             * throw ZkInterruptedException and skip ZkConnection#close()
             */
            if (_connection != null) {
                try {
                    /**
                     * ZkInterruptedException#construct() honors InterruptedException by calling
                     * Thread.currentThread().interrupt(); clear it first, so we can safely close the
                     * zk-connection
                     */
                    Thread.interrupted();
                    _connection.close();
                    /**
                     * restore interrupted status of current thread
                     */
                    Thread.currentThread().interrupt();
                } catch (InterruptedException e1) {
                    throw new ZkInterruptedException(e1);
                }
            }
        } finally {
            getEventLock().unlock();
            if (_monitor != null) {
                _monitor.unregister();
            }
            LOG.info("Closed zkclient");
        }
    }

后来在[github issue](https://github.com/sgroschupf/zkclient/issues/34)里也找到有zk用户提出该问题,zkclient代码还是暂时不动吧.

建议用户close前添加sleep, 避开该问题,同时new client采用单例,毕竟一个应用与zk 保持一个client就可以,与redis的连接并发多个是没问题的.

后续思考
----------
思考该用户场景,为什么其他用户未碰到

主要是大多数场景都是new client后,一直使用该client,不使用时进程也要结束了.而该用户场景是进程中启动一个任务线程,线程中new client->操作,然后终止线程,自行释放资源并调用close(),然后不断循环该过程导致web监控到该异常.

再思考为什么`r2m_client`的作者会把捕获那个异常注释掉,估计是觉得该操作失败的异常对于r2m集群使用没有影响,不要中断后续处理,该异常在任务结束时会频繁抛出,而且如果打印出信息会引起用户疑问,而作者考虑到异常未关闭的zk连接在进程结束后也会释放掉.

R2M系统介绍
----------
R2M系统是基于开源的Redis cluster(Redis 3.0以上版本)研发的高性能的分布式缓存系统,京东金融的大部分缓存服务都是跑在R2M上,已经平稳的保障了多次双十一和618大促,性能,可靠性和数据一致性得到了充分的验证.R2M在满足业务要求的同时,也一直在优化运维的需求,提供Web化的一键部署,数据平衡（Rebalance）,水平扩容,数据迁移,监控告警,在线命令行管理,多机房切换等一系列功能.R2M系统架构上保持精简,不依赖其他团队的组件,可以在新环境快速进行独立部署.

[R2M系统详情查看](https://studygolang.com/articles/10958?fr=sidebar)

