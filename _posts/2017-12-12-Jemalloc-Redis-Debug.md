---
layout: post
title:  基于Jemalloc下Redis内存分析
date:   2017-12-12 00:12:52 +0800
categories: Dev
tag: Redis
---

* content
{:toc}

背景
----------
线上某业务Redis集群需要迁移到[SWAPDB](#swapdb)，预先做一次线下模拟场景，主要评估内存节约情况(本次迁移主要目的).

过程中碰到与预期不符的内存占用问题，经过分析对基于`Jemalloc`下Redis的内存使用有了更加深入理解.

注：以下场景及分析均基于64bit系统.

场景模拟
----------
目标业务Redis 3.2集群为三主三从，每个主节点使用内存约为3.4GB，从节点约为3.34GB(相差`repl-backlog-size`64MB大小)，Key数量为1500W，均为hash类型，且value格式均为：

	redis> hgetall 52acb36a653d48629d6cdc667a9f12de
	(0)123456789012345-123456789012
	(1)1

基于以上信息模拟如下数据：

	for i in range(0， 45000000):
	#key为32为uuid字符串，实际集群使用的key是全随机16进制字符串，但经过对比此差异对结果并无影响
		key = str(uuid.uuid4()).replace('-'，'')
	#r为python 支持cluster接口
		r.hset(key， "123456789012345-123456789012"， 1)

启动与业务集群一致的Redis 3.2三主三从集群，并将以上数据写入，结果如下：

	used_memory:3650444616
	used_memory_human:3.40G

与线上内存占用一致，数据模拟正确.

启动SWAPDB三主三从集群，并将以上数据写入，通过设置`ssdb-transfer-lower-limit 100`禁止转存，此时期望内存与Redis 3.2集群一直，结果如下：

	used_memory:3126138896
	used_memory_human:2.91G

内存节省约490MB...... 那么问题来了:

__SWAPDB是通过转存冷数据的value到SSD中节约内存的，在未转存时应与redis内存保持一致，为什么此时节约490MB内存?__

考虑到SWAPDB是基于Redis 4.0做的改动，先查看是否由于Redis版本升级导致的，启动原生Redis 4.0集群对比：

	used_memory:3125854232
	used_memory_human:2.91G

可以看到其与SWAPDB是一致的，故该处节约490MB内存是Redis 4.0版本升级导致的.

Redis内存优化分析
----------
在[Redis 4.0 日志](https://raw.githubusercontent.com/antirez/redis/4.0/00-RELEASENOTES)中搜索关于Memory优化的commit，未找到相关说明.

考虑到Redis可能对key-value DB存储做出过优化，启动单实例(带一个从节点)写入数据，结果Redis 3.2/ Redis 4.0均使用内存大约为1.91GB.

即key-value DB存储处起码针对此次hash类型key-value未有内存优化的升级.

只能是在集群相关处有内存优化可能.而且只对比单实例与集群节点内存差距也很大：

	Redis 4.0: 2.91 - 1.91 GB - 64MB = 960MB
	Redis 3.2: 3.40 - 1.91 GB - 64MB = 1.43GB
	注： 减去64MB是主节点多出来的repl-backlog-size大小

内存差距如此大肯定是跟记录1500W个key信息有关，和集群相关的数据结构包括：`clusterNode/clusterLink/clusterState`，数据结构源码比较简单，最后在Redis 4.0 代码的clusterState结构体里找到：

	uint64_t slots_keys_count[CLUSTER_SLOTS];
    rax *slots_to_keys;

`Rax tree`确实是作者新加入的数据结构，之前看过文章，了解了下其结构，但没看到作者说其在redis哪块使用，也没去代码里找，在[Redis 4.0 日志](https://raw.githubusercontent.com/antirez/redis/4.0/00-RELEASENOTES)中提到，说是为了修复集群慢的问题：

	* A new data structure， the radix tree (rax.c) was introduced into Redis in
	  order to fix a major Redis Cluster slowdown. (Salvatore Sanfilippo)

对比Redis 3.2代码：

	zskiplist *slots_to_keys;

从名称上看得出来这是记录slot和key的映射关系，查看代码其调用函数`getKeysInSlot()`遍历该结构查询对应slot有哪些key，所以结构里存储了1500W key的信息占用了大量内存.

zskiplist内存分析
----------
`zskiplist`由`zskiplistNode`链接而成，具体介绍见后续[参考](#zskiplist)，Redis 3.2中`zskiplistNode`定义：

	typedef struct zskiplistNode {
		robj *obj;    //8 bytes + obj len，Redis 4.0中改成sds ele，更节约内存
		double score; //8 bytes
		struct zskiplistNode *backward; //8 bytes
		struct zskiplistLevel {
			struct zskiplistNode *forward; //8 bytes
			unsigned int span; //4 bytes
		} level[];  //n level
	} zskiplistNode;

key对应的`hash slot`为score，key存储在obj对象之中.

查看robj及sds定义：

	typedef struct redisObject {
		unsigned type:4;
		unsigned encoding:4;
		unsigned lru:LRU_BITS; //4 bytes
		int refcount; //4 bytes
		void *ptr; //8 bytes + sds len (createEmbeddedStringObject， sdshdr8)
	} robj;

	struct __attribute__ ((__packed__)) sdshdr8 {
		uint8_t len; //1 byte
		uint8_t alloc; //1 byte
		unsigned char flags; //1 byte
		char buf[]; //key len
	};

如上，每个`zskiplistNode`占用内存为

    8+obj len +8+8+(8+4)*n=24+12*n+4+4+8+sds len=40+12*n+1+1+1+key len+1=76+12*n bytes

Redis 3.2 集群节点比单实例节点大1.43GB， 平均到每个key大约为95.3 Bytes. n约为1.608，即`zskiplistNode`的level平均为1.608.

ZskiplistNode Level计算
----------
    
    int zslRandomLevel(void) {
        int level = 1;
        while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF)) //ZSKIPLIST_P==0.25
            level += 1;
        return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;//ZSKIPLIST_MAXLEVEL==32
    }
    
该函数返回level期望为1.333. 基于该值算出1500W个ziplistnode占用内存为1.38GB左右.与前面的1.43GB差距为50MB，考虑到集群其他一些结构也会占用少量内存，并且Jemalloc内存分配内存是按block分配的， 一整块一整块分配，也不可能完全精确计算，该误差可认为在正常范围内.

Rax tree 结构
----------
Redis 4.0 集群节点比单实例节点大1GB， 平均到每个key大约为67 Bytes.

	typedef struct rax {
		raxNode *head;
		uint64_t numele;
		uint64_t numnodes;
	} rax;

	typedef struct raxNode {
		uint32_t iskey:1;     /* Does this node contain a key? */
		uint32_t isnull:1;    /* Associated value is NULL (don't store it). */
		uint32_t iscompr:1;   /* Node is compressed. */
		uint32_t size:29;     /* Number of children， or compressed string len. */
		unsigned char data[];
	} raxNode;

从raxNode的结构来看，由于其每个节点大小都不固定，不能像分析zskiplist内存一样每个Node占用内存基本相同来计算总体.

先对`rax tree`单独分析，下载rax项目[rax 源码](https://github.com/antirez/rax.git)，将以下代码加入`rax-test.c`中:

	char *random_uuid( char buf[33] )
	{
		char *p = buf;
		int n;
		for( n = 0; n < 32; ++n )
		{
			int b = rand()%16;
			sprintf(p， "%x"， b);
			p += 1;
		}
		*p = 0;
		return buf;
	}
	void uuidMemTest(){
		rax *t = raxNew();
		char buf[64]， guid[33];
		unsigned int hashslot;
		int len = 34;   //2 bytes(hashslot)+32 bytes(keylen)
		for (int i = 0; i < 45000000; i++) {
			random_uuid(guid);
			hashslot = i%16384;   //slot信息占两个字节加入到key头部
			if(hashslot>5460) continue;

			buf[0] = (hashslot >> 8) & 0xff;
			buf[1] = hashslot & 0xff;

			sprintf(buf+2，"%s"，guid);

			raxInsert(t，(unsigned char*)buf，len，NULL，NULL);
		}
	}

在`main`函数中运行`uuidMemTest()`，运行完毕后查看程序占用内存，结果如下：

	t->numel == 15001367， t->numnodes=35066975， 内存使用 1.482g

`t->numel`为插入key的数量，`t->numnodes`为`rax tree`中建立的raxNode数量，内存使用为1.482g，远超过了预期的960MB.

查看该项目中`rax_malloc.h`文件中默认使用的是libc内存分配器，而Redis 3.2和4.0均使用的是jemalloc.所以先使用jemalloc替换该项目中的libc.

Jemalloc替换
----------
替换步骤如下
1. rax\_malloc.h文件中:

	malloc->je_malloc
	realloc->je_realloc
	free->je_free

2. rax.h文件中加入`#include <jemalloc/jemalloc.h>`
3. github上下载[jemalloc](https://github.com/jemalloc/jemalloc/releases)，解压后进入目录执行:

	./configure --with-jemalloc-prefix=je_ --prefix=/usr/local/jemalloc
	make -j8 && make install
	echo /usr/local/jemalloc/lib >> /etc/ld.so.conf
	ldconfig

4. Makefile文件中编译rax-test目标命令后添加`-L/usr/local/jemalloc/lib -ljemalloc`

替换完成，编译运行rax-test即可.

Rax tree with Jemalloc 内存
----------
替换成`jemalloc`后运行<span id="uuid result">结果</span>:

	t->numel == 15001367， t->numnodes=35066975， 内存使用992.5MB.

虽然场景中redis集群节点只多了960MB大小内存，与单独插入1500W keys的`rax tree`占用大小992.5MB有32MB差距，但考虑到`Jemalloc`内存分配内存是按block分配的， 一整块一整块分配，也不可能完全精确计算，该误差在正常范围内，可认为Redis 4.0集群节点多出来的内存都是Rax tree占用的.

接下来分析下1500W keys的`rax tree`如何使用的900+MB内存.

数据看起来有点复杂......简化起见先对写入数据为(000000-999999)的`rax tree`进行分析：

	for (int i = 0; i <= 1000000; i++) {
        sprintf(buf， "%06d"， i);
        raxInsert(t， (unsigned char *) buf， 6， NULL， NULL);
    }

运行上面代码写入数据后结果:

	t->numel == 1000000， t->numnodes=1111111， 内存使用24280 kb.(写入前为5848 kb)
	即100w个元素(000000-999999)`rax tree`占用了24280 - 5848 kb = 18432 kb

从代码数据结构分析(详情见[参考](#rax))`rax tree`占用，100w个元素(000000-999999)的`rax tree`结构如下：

    [node header][0-9](10 ptr)   (94 bytes: 4 bytes:header+ 10 bytes:0 ~ 9 字符 + 80 bytes:10个bit64位指针)
    |
    [node header][0-9](10 ptr)   * 10 940bytes
    |
    [node header][0-9](10 ptr)   * 10*10 9400bytes
    |
    [node header][0-9](10 ptr)   * 10 * 10 *10   94000bytes
    |
    [node header][0-9](10 ptr)   * 10 * 10 * 10 * 10  940000bytes
    |
    [node header][0-9](10 ptr)   * 10 * 10 * 10 * 10 * 10    = 100，000  9400000bytes
    |
    [node header] *100w 400w bytes
    每个元素指向一个空 data 的raxNode，指示该节点是一个key  (4bytes) 100w个插入元素

所以数据(000000-999999)插入后内存应占用以上之和又相当于

    t->numnodes * 4 (每个node head大小) + (t->numnodes-100w) * 90 (每个节点字符和其指针大小)  = 14444434 bytes.

很明显与实际占用18432 kb = 18874368 bytes差别很大.

内存差异原因
----------
尝试通过valgrind的memcheck找下是否有内存泄漏问题，运行如下

    valgrind --track-origins=yes --suppressions=./valgrind.sup --show-reachable=no --show-possibly-lost=no --leak-check=full ./rax-test

打印如下：

	==16686== 8 bytes in 1 blocks are definitely lost in loss record 1 of 3
	==16686==    at 0x514864A: je_malloc (jemalloc.c:1441)
	==16686==    by 0x403AAB: raxNewNode (rax.c:145)

期望raxNewNode()应该是分配4 bytes(node head大小).却分配了8 bytes，debug查看调用`je_malloc`处也是4.

原因是jemalloc最小分配内存单元是8bytes的分配机制导致，写程序测试多次jemalloc场景并使用命令top显示验证结果如下:

	14431 zj   20   0   19888   4688   2432 S   0.0  0.0   0:00.02 demo   0 次分配程序初始内存
	14471 zj   20   0   30128  15032   2540 S   1.7  0.1   0:00.05 demo   1000*1024次分配4bytes
	14503 zj   20   0   30128  14956   2464 S   1.0  0.1   0:00.04 demo   每次8 bytes
	14540 zj   20   0   38320  23136   2452 S   1.0  0.1   0:00.05 demo   每次10 bytes
	14571 zj   20   0   38320  23108   2424 S   2.0  0.1   0:00.06 demo   每次16 bytes

可以看到每次分配4bytes和8字节是一样的，每次分配10bytes和16bytes也是一样的.

所以`rax tree`每个node虽然size是4bytes，但是每次`jemalloc`需分配8字节.

故`rax tree`内存实际由`jemalloc`分配

    t->numnodes*8 (每个node head) + (t->numnodes-100w)*90 (每个节点字符和其指针)=18888878 bytes~=18874368 bytes

注:`jemalloc`的分配机制也影响到前面的分析，计算出来的内存无法完全与实际分配的内存一致，只是在此处由于数量巨大，才明显出现差异.

1500w uuid rax tree 内存分析
----------
见Rax tree with Jemalloc 内存[运行结果](#uuid result):

    t->numel == 15001367， t->numnodes=35066975， 内存使用992.5MB.

即随机数1500W个uuid插入后，占用3500W个nodes，内存大约1GB.

每个uuid key插入时，其slot信息(2个字节)加入到其头部:

	void slotToKeyUpdateKey(robj *key, int add) {
		int ret;
		unsigned int hashslot = keyHashSlot(key->ptr,sdslen(key->ptr));
		unsigned char buf[64];
		unsigned char *indexed = buf;
		size_t keylen = sdslen(key->ptr);

		if (!server.swap_mode) server.cluster->slots_keys_count[hashslot] += add ? 1 : -1;
		if (keylen+2 > 64) indexed = zmalloc(keylen+2);
		indexed[0] = (hashslot >> 8) & 0xff;
		indexed[1] = hashslot & 0xff;
		memcpy(indexed+2,key->ptr,keylen);
		if (add) {
			ret = raxInsert(server.cluster->slots_to_keys,indexed,keylen+2,NULL,NULL);
			if (server.swap_mode && ret == 1)
				server.cluster->slots_keys_count[hashslot] += 1;
		} else {
			ret = raxRemove(server.cluster->slots_to_keys,indexed,keylen+2,NULL);
			if (server.swap_mode && ret == 1)
				server.cluster->slots_keys_count[hashslot] += -1;
		}
		if (indexed != buf) zfree(indexed);
	}
	
slot `0 ~ 5460`即十六进制的`0x0~0x1554`,则第一个节点为0~21,第二个节点为0~255,其中21对应的下层节点范围为`0x0-0x54(84)`,此时树结构大致如下:

	1 [node header][0-20/21](22 ptr)     206 bytes: 8+22+22*8
	|
	2 [node header][0-255](256 ptr) *21 ---- [node header][0-84](85 ptr) *1  3085 bytes: 8+256+256*8 + 8+85+85*8
	|
	3 [node header][0-9a-f](16 ptr) * 5461  152bytes*5461 每个节点命中1500W/5461次
	|
	4 [node header][0-9a-f](16 ptr) * 5461 *16 152bytes*5461*16 每个节点命中1500W/5461/16次
	|
	5 [node header][0-9a-f](16 ptr) * 5461 *16 *16 152bytes*5461*16*16 每个节点命中1500W/5461/16/16~=10次
	|
	6 这一层node不好估计...
    :
    :
    35  每个ptr指向一个空data 的raxNode，指示该节点是一个key  (8bytes) 1500w个插入元素

前面1-5层由于命中次数多，则其后续字符覆盖`[0-9a-f]`概率很高，故可以认为每个节点均拥有16个孩子节点.

到第5层时，由于每次父节点命中平均只有10次，故其后续字符部分可能只命中一次，部分可能也能命中几次，只命中一次的节点则为compress nodes 
`[node header][27 chars](1 ptr)`，重复命中的节点在第6层仍为普通节点，在第7层为compress nodes `[node header][26 chars](1 ptr)`.

需分析命中一次和多次节点数的期望，即10个`[0-9a-f]`的字符重复一次和多次的期望.[**TODO**]()

期望分析比较复杂，暂时没有解决方案，先分析两种极端情况:

1. 极端情况下如果这10次每次都不一样的字符，则第6层开始均为1500W个compress nodes:  

    - `[node header][27 chars](1 ptr) * 1500W    (8+27+8)*1500W`

    - 则该情况下

    - `t->numnodes=1+22+5461+5461*16+5461*16^2+1500W+1500W=31490876`
    - `206+3085+152*(5461+5461*16+5461*16^2)+43*1500W+8*1500W=991612947 bytes ~= 945.7MB`

2. 另一种极端情况为这10次每次命中同一种字符，到了第7层每次命中不一样的字符，第7层开始均为1500W个compress nodes

    - 第6层:`[node header][0-f](10 ptr) * 5461*16*16         (8+10+10*8)*5461*16*16`
    - 第7层:`[node header][26 chars](1 ptr) * 1500W    (8+26+8)*1500W`

    - 则该情况下

    - `t->numnodes=1+22+5461+5461*16+5461*16^2+1500W+1500W+5461*16*16=32888892`
    - `206+3085+152*5461*(1+16+16^2)+42*1500W+8*1500W+98*5461*16*16=1113618515 bytes ~= 1062MB`

从以上两种情况结果看，与实际运行的结果`(节点数和内存)`比较接近.

总结
----------
集群模式下，记录slot和key的映射关系的`slots_to_keys`会占用比较可观的内存，Redis 4.0使用rax tree替代zskiplist结构存储不仅解决了其日志中`Redis Cluster slowdown`问题.

同时节约了一定量的内存，本场景1500W个uuid key相应内存使用从1.43GB降至960MB.

最后从对`jemalloc`的分析可以看出，在每次分配每个node`(4 bytes)`时会分配8 bytes的内存块，此处多分配`35066975(节点数)*4 bytes`=133.8MB，也许`rax tree`还有优化空间.

参考
----------
<span id="swapdb">[swapdb - 基于redis改造的冷热数据存储分离数据库](https://github.com/JRHZRD/swapdb)</span>

<span id="rax">[Rax, an ANSI C radix tree implementation](https://github.com/antirez/rax)</span>

<span id="zskiplist">[zskiplist - Redis内部数据结构详解之跳跃表](http://redisbook.readthedocs.io/en/latest/internal-datastruct/skiplist.html)</span>

[jemalloc和内存管里](http://www.cnblogs.com/gaoxing/p/4253833.html)

[jemalloc在linux上从安装到使用](http://blog.csdn.net/xiaofei_hah0000/article/details/52214592)
