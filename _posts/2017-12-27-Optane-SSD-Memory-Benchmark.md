---
layout: post
title:  Optane SSD虚拟内存Redis性能测试
date:   2017-12-27 11:26:34 +0800
categories: Test
tag: Redis
---

* content
{:toc}

背景
----------
对大规模使用redis作为缓存的项目，Intel提供了一种降低成本的方案: 使用Intel Optane SSD，应用其Memory Drive技术，可大大扩展系统内存，据说实际使用能达到当前主流DDR4内存的70-80%.

价格来源于京东商城2017-12-27数据:

    英特尔（Intel） Optane DC P4800X傲腾 数据中心PCI-E NVMe 375GB价格为20000 RMB
    戴尔（DELL） 全新盒装服务器 ECC 内存条 16G丨DDR4 2133 Mhz UDIMM 价格为1680 RMB

以扩展到512G内存价格对比:

    64GB DDR4 + 2*320GB SSD = 46720 RMB //Intel产品手册说明需要这样扩展
    512GB DDR4 = 53760 RMB

价格优势很不明显...可能规模销量上去了会降价吧...不过最近内存的价格也涨了近一倍...

不管怎么，总算多了个选择...

据此对其扩展后的内存做一次性能测试，期望测试出其相当于原生内存性能的xx%.

场景模拟
----------
机器实际内存由8条16G DDR4组成，共128G，加入一块320G 的Optane SSD(官方建议每个CPU外接一个SSD)并执行相关安装设置后，内存扩展生效，变成342G.（如果原内存是192G，可扩展成512G)

对比性能测试最好能再找一台320G DRAM-only其他配置(CPU/操作系统及版本等)与实验机器一致的.然而实验环境有限，只有先用这一台已由相关人员配置好的机器进行测试.

原生Redis 4.0，使用redis-benchmark测试.

猜想Redis会优先使用128G原生内存，使用完才开始使用SSD扩展的内存.

所以需要对比使用内存小于128G时（原生内存性能）与使用内存大于128G时（扩展内存性能）.

故测试场景如下：

修改`redis-benchmark.c`增加个`-R`选项，能依次遍历指定范围(0-2000w)数值key，即`-t set -R 20000000`时能依次写入`key:000000000000-key:000019999999`.

1. 启动redis-server 端口为9001，禁止持久化.

2. set 2000W个key value=1K

    redis-benchmark -R 20000000 -n 20000000 -t set -d 1000 -p 9001  (19W tps)

3. 打入大约140G数据

    redis-benchmark -P 100 -r 1000000000 -n 14000000 -t hset -d 10000 -p 9001

4. 启动redis-server 端口为9002，禁止持久化

5. set 2000W个key value=1K

    redis-benchmark -R 20000000 -n 20000000 -t set -d 1000 -p 9002  (13.5w tps)

再次执行以上步骤结果一致，故得出结论扩展内存性能降至13.5/19~=70%左右.

对结论进行确认
----------
得出以上结论后，由于不明白其扩展内存原理，据说使用了冷热数据分离存储，不敢确定上述思路是否正确.

听说热数据仍使用原生内存，冷数据交换到optane ssd中，故尝试测试出该场景，以确认该来源不靠谱信息.

从步骤5开始，反复访问范围在10w内的数据，以期望其变热，tps从13.5w上升至19w.

    redis-benchmark -r 100000 -n 10000000 -t set -d 1000 -p 9002

然后就发现一个诡异问题，每次运行时tps不稳定，一会19w，一会13.5w.

问题复现
----------
概率性的问题不好分析，需要区分出其tps高和低时的条件或场景.

尝试kill掉redis-server清空内存，再次启动新redis-server，重复执行几次benchmark还是反反复复，结果不确定，有点像三体中智子对粒子对撞机的干扰.

换台对撞机B，使用自用的机器运行多次benchmark，结果很稳定.

再换台对撞机C，使用公用测试环境机器，运行多次benchmark，结果也不稳定，tps浮动40%左右.

考虑到对撞机C上有很多进程在运行，可能抢占CPU影响redis性能，top查看也显示48个CPU使用率波动比较厉害.

通过对撞机C想起CPU这个因素来了，但是实验机器对撞机A上没有别的进程在运行，top查看其32个CPU使用率均为0.运行benchmark时也均为0.？！

    %Cpu0  :  0.0 us，  0.0 sy，  0.0 ni，100.0 id，  0.0 wa，  0.0 hi，  0.0 si，  0.0 st
    %Cpu1  :  0.0 us，  0.0 sy，  0.0 ni，100.0 id，  0.0 wa，  0.0 hi，  0.0 si，  0.0 st
    ...
    %Cpu31  :  0.0 us，  0.0 sy，  0.0 ni，100.0 id，  0.0 wa，  0.0 hi，  0.0 si，  0.0 st

    10649 root      20   0 44.035g 0.043t   1596 R 100.0 12.8   1:18.33 redis-server                                                      
    10660 root      20   0  212368  77160    912 S  15.6  0.0   0:10.39 redis-benchmark 


这是显示有问题吧，还是使用pidstat看看进程都运行在哪个cpu.

    [root@dell-test ~]# pidstat | grep 10649
    12:43:01 AM     0     10649    0.01    0.02    0.00    0.03    16  redis-server

    [root@dell-test ~]# pidstat | grep 10660
    12:44:25 AM     0     10660    0.00    0.03    0.00    0.03    17  redis-benchmark

redis-server运行在CPU16，redis-benchmark运行在CPU17.

再次运行benchmark，redis-server还是CPU16，而redis-benchmark却运行在CPU25了，这也是很好理解的，毕竟redis-benchmark进程重新启动了，
系统重新给其指定CPU.

重复运行了几次发现当redis-benchmark运行在特定CPU如0，16，17，18时，tps能达到19w，而在CPU25时，tps降至13.5w，问题稳定复现.

问题分析
----------
首先已知CPU 0性能正常(在前面运行中能达到高tps)，先指定redis-server运行在CPU 0上

执行以下命令统计哪些cpu性能差，重复执行人次以上

    [root@dell-test ~]# for i in {1..31};do echo "======cpu $i======";taskset -c $i redis-benchmark -t set -p 9001 -q;done

结果显示CPU 8-16以及24-31性能差，tps在13w左右，数量刚好是总CPU的一半.

统计对撞机B中性能均一致.

统计对撞机C中性能差的CPU为0-11，24-35共24个，数量也刚好是总CPU的一半.

测试结果也显示当redis-server或redis-benchmark进程有一个运行在性能差的CPU上，其性能就变差了.

测试两个进程同在性能差的CPU上，如一个在CPU8，一个在CPU9上，然后发现性能变好了...

这时候反应过来了，其实这32个CPU中没有性能好坏之分，只是因为两个进程运行在跨NUMA node的cpu上了.

NUMA架构
----------
非统一内存访问架构（英语：Non-uniform memory access，简称NUMA）是一种为多处理器的电脑设计的内存，内存访问时间取决于内存相对于处理器的位置.在NUMA下，处理器访问它自己的本地内存的速度比非本地内存（内存位于另一个处理器，或者是处理器之间共享的内存）快一些.

实验机器lscpu显示

    NUMA node0 CPU(s):     0-7，16-23
    NUMA node1 CPU(s):     8-15，24-31

当两个进程运行的CPU属于不同NUMA node时，跨NUMA node通信更慢导致tps下降.

查看当前redis-server运行CPU

	[root@dell-test ~]# pidstat | grep redis-server
	03:41:11 AM     0     10742    0.06    0.20    0.00    0.26     0  redis-server

循环运行redis-benchmark

	while true;do redis-benchmark -t set -p 9001 -n 200000;done
	
在另一个窗口持续监控redis-benchmark运行CPU情况，此时redis-server始终运行在CPU0上（因为没有别的进程来抢占）
while true;do pidstat | grep redis-benchmark;sleep 1;done
结果

    [root@dell-test ~]# while true;do pidstat | grep redis-benchmark;sleep 1;done
    03:43:21 AM     0     12797    0.00    0.00    0.00    0.00    17  redis-benchmark
    03:43:22 AM     0     12801    0.00    0.00    0.00    0.00    25  redis-benchmark
    03:43:23 AM     0     12801    0.00    0.00    0.00    0.00    25  redis-benchmark
    03:43:24 AM     0     12808    0.00    0.00    0.00    0.00    17  redis-benchmark
    03:43:25 AM     0     12812    0.00    0.00    0.00    0.00    17  redis-benchmark
    03:43:26 AM     0     12816    0.00    0.00    0.00    0.00    17  redis-benchmark
    03:43:27 AM     0     12820    0.00    0.00    0.00    0.00    17  redis-benchmark
    03:43:28 AM     0     12824    0.00    0.00    0.00    0.00    17  redis-benchmark
    03:43:29 AM     0     12828    0.00    0.00    0.00    0.00    17  redis-benchmark
    03:43:30 AM     0     12832    0.00    0.00    0.00    0.00    25  redis-benchmark
    03:43:31 AM     0     12836    0.00    0.00    0.00    0.00    25  redis-benchmark
    03:43:32 AM     0     12836    0.00    0.00    0.00    0.00    25  redis-benchmark

redis-benchmark随机在CPU17和25间运行，当在CPU 17运行时，与CPU 0属于同NUMA node，此时tps达到19w，当在CPU 25运行时，与CPU 0不属于同NUMA node，此时tps降为13.5w.

    [root@dell-test ~]# lscpu
    Architecture:          x86_64
    CPU op-mode(s):        32-bit， 64-bit
    Byte Order:            Little Endian
    CPU(s):                32
    On-line CPU(s) list:   0-31
    Thread(s) per core:    2
    Core(s) per socket:    8
    Socket(s):             2
    NUMA node(s):          3
    Vendor ID:             GenuineIntel
    CPU family:            6
    Model:                 63
    Model name:            Intel(R) Xeon(R) CPU E5-2640 v3 @ 2.60GHz
    Stepping:              2
    CPU MHz:               1522.320
    BogoMIPS:              5199.27
    Virtualization:        VT-x
    L1d cache:             32K
    L1i cache:             32K
    L2 cache:              256K
    L3 cache:              20480K
    NUMA node0 CPU(s):     0-7，16-23
    NUMA node1 CPU(s):     8-15，24-31
    NUMA node2 CPU(s):

从lscpu的信息看其实实验机器是有两个CPU， 每个CPU 8核，每个核虚拟为2个逻辑核心，每个CPU对应一个NUMA node，有各自的存储控制器.

指定cpu运行结果
----------
指定两个进程(redis-benchmark和redis-server)运行在同一NUMA node的两个非CPU0的空闲CPU上，重新运行测试，运行三次，结果如下

Test No.| Read/Write| TPS(us) | TP999(us) | Maxi Time(us)
-|-|-|-|-
1 | BSet | 187451.97 | 4 | 53
  | BGet | 217183.56 | 2 | 4
  | ASet | 175327.86 | 3 | 95
  | AGet | 204712.48 | 2 | 3
2 | BSet | 193348.80 | 3 | 40
  | BGet | 219401.70 | 2 | 4 
  | ASet | 171744.58 | 3 | 123
  | AGet | 172868.31 | 2 | 3
3 | BSet | 192226.38 | 3 | 30
  | BGet | 217644.44 | 2 | 4
  | ASet | 169950.97 | 3 | 103
  | AGet | 165436.92 | 2 | 3

注: BGet/BGet为内存小于128G时结果，AGet/AGet为内存大于128G时结果.

从结果看使用内存超过原生内存时Set tps性能降低约6-10%. TP999保持一致，最大延迟时间差距比较大，大概2-3倍，而get tps性能降低约6-24%，每次运行结果浮动很大，延迟却差不多

对本次性能测试存怀疑态度，留待使用一台配置相同，全内存DRAM的机器与其进行对比

总结
----------
在此次对Intel Optane SSD虚拟内存性能测试过程中，了解到Intel该SSD虚拟化内存的黑科技，学会及应用了linux两个命令taskset以及pidstat

特别是了解了CPU的NUMA架构以及感受到了其对性能的影响，在以后性能测试中注意该问题

参考
----------

[扩展内存: intel-memory-drive-technology](https://www.intel.com/content/www/us/en/software/intel-memory-drive-technology.html)

[Linux 运行进程实时监控pidstat命令详解](http://www.cnblogs.com/ggjucheng/archive/2013/01/13/2858874.html)

[taskset: 让进程运行在指定的CPU](http://www.cnblogs.com/edwardlost/archive/2010/10/23/1858991.html)

[NUMA架构的CPU](http://cenalulu.github.io/linux/numa/)
