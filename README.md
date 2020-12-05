## 1 赛题介绍

### 1.1 赛题

Tair是阿里云自研的云原生内存数据库，接口兼容开源Redis/Memcache。在阿里巴巴集团内和阿里云上提供了缓存服务和高性能存储服务，追求极致的性能和稳定性。全新英特尔® 傲腾™ 数据中心级持久内存重新定义了传统的架构在内存密集型工作模式，具有突破性的性能水平，同时具有持久化能力，Aliyun弹性计算服务首次（全球首家）在神龙裸金属服务器上引入傲腾持久内存，撘配阿里云官方提供的Linux操作系统镜像Aliyun Linux，深度优化完善支持，为客户提供安全、稳定、高性能的体验。本题结合Tair基于神龙非易失性内存增强型裸金属实例和Aliyun Linux操作系统，探索新介质和新软件系统上极致的持久化和性能。参赛者在充分认知Aep硬件特质特性的情况下设计最优的数据结构。

### 1.2 赛题描述

热点是阿里巴巴双十一洪峰流量下最重要的一个问题，双十一淘宝首页上的一件热门商品的访问TPS在千万级别。这样的热点Key，在分布式的场景下也只会路由到其中的一台服务器上。解决这个问题的方法有Tair目前提供的热点散列机制，同时在单节点能力上需要做更多的增强。

本题设计一个基于傲腾持久化内存(Aep)的KeyValue单机引擎，支持Set和Get的数据接口，同时对于热点访问具有良好的性能。

**语言限定**：C/C++
**初赛资源情况**：内存4G、持久化内存(Aep)74G 、cpu16核
**复赛资源情况**：内存8G（再加320M用户空间）、持久化内存(Aep)74G 、cpu16核

> 赛题详细见：https://code.aliyun.com/db_contest_2nd/tair-contest/tree/master </br>
> 比赛链接：https://tianchi.aliyun.com/competition/entrance/531820/information </br>
> 英特尔傲腾持久内存综述：https://tianchi.aliyun.com/course/video?liveId=41202

## 2 初赛

### 2.1 评测逻辑

引擎使用的内存和持久化内存限制在 4G Dram和 74G Aep，无持久化要求。

**评测分为两个阶段**
* 正确性评测（不计入耗时）：开启16个线程并发写入一定量KV对象，并验证读取和更新后的正确性
* 性能评测：16个线程并发调用48M个Key大小为16Bytes，Value大小为80Bytes的KV对象，接着以95：5的读写比例访问调用48M次。其中读访问具有热点的特征，大部分的读访问集中在少量的Key上面

### 2.2 赛题分析

* Set阶段
    * 只有少量更新操作
    * 总的key的数量大约有768M，总写入的kv数据量有72G，是小于Aep74G
    * 无持久化要求
* Get阶段：
    * 读具有热点Key特征
    * 该阶段的set都是key更新操作

### 2.3 方案

#### 2.3.1 架构

整个方案的架构如下图所示。
 

* Buffer池：Set阶段每个线程先写4M块buffer，buffer写满之后，再提交给异步线程落盘
* hash索引：对key进行hash分桶
* 缓存：Get阶段对读进行缓存

#### 2.3.2 数据结构

##### 1 索引和KV的数据结构

* 同一个hash桶内kv数据之间使用链表存储，insert使用头插法，insert的时间复杂度就是O(1)
* key和value都是定长，所以pre_ptr记的是写入的kv个数，只需4个字节表示
* 由于key是均匀分布的，所以使用的是key的前8个字节的低28位作为hash值

##### 2 缓存数据结构

### 2.4 方案效果

* 内存使用情况
    * hash索引：1G
    * 读缓存：16M
    * buffer池：128M
    * kv写缓存：3.12G
* 最优耗时：38.627s（set阶段24.5s，get阶段14.1s）
* 最终排名：第一名

### 2.5 其它优化

* Init阶段优化：
    * 多线程分段初始化hash索引
* Set阶段优化
    * update采用追加插入方式，Get阶段的Set使用更新方式
    * hash索引更新使用cas无锁操作
    * 利用多余内存(3G+)来缓存头部写入KV
* Get阶段优化
    * 每轮读写后清空缓存
    * 初赛没有持久化要求，所以该阶段的update操作更新缓存成功就不更新aep

## 3 复赛

### 3.1 评测逻辑

**评测分为三个阶段：**
* 正确性评测：16个线程并发写入KV对象（Key固定大小16bytes，Value大小范围在80-1024bytes），然后验证读取和更新后的正确性
* 持久化评测：模拟断电场景，验证写入数据断电恢复后不受影响。该阶段不提供日志
* 性能评测（计入成绩）：
    * Set阶段：16个线程并发调用24M次Set操作，并选择性读取验证
    * Get阶段：16个线程以75%：25%的读写比例调用24M次，其中读符合热点Key特征。该阶段有10轮测试，取最慢一次的结果作为成绩

**性能阶段的数据特点：**

会保证任意时刻数据的value部分长度和不超过50G

纯写入的24M次操作中

写入val长度

80-128bytes

129-256bytes

257-512bytes

513-1024bytes

占比

55%

25%

15%

5%

总数据量

75G左右

Get阶段中的所有Set操作的Value长度均不超过128bytes并且全部是更新操作

**要求：**
* 对于Insert
    * 已经返回的key，要求断电后一定可以读到数据。
    * 尚未返回的key，要求数据写入做到原子性，在恢复后要么返回NotFound，要么返回正确的value。

* 对于Update
    * 已经返回的key，要求断电后一定可以读到新数据。
    * 尚未返回的key，要求更新做到原子性，在恢复后要么返回旧的value，要么返回新的value。

### 3.2 赛题分析

* Set阶段：
    * 持久化要求：数据断电后不丢失，无法再使用Buffer来批量落盘，每次Set写盘都需要刷盘才能保证断电后数据不丢失
    * Aep容量不够：Aep（64G）的容量小于总写入KV数据量（75G），需要有一定的回收机制
    * Key的数量：Key数量大概有220M ，性能阶段总set次数是384M，有大概164M次set是更新操作
* Get阶段：
    * 读具有热点Key特征，肯定要使用缓存
    * Get阶段计分是是取的10轮中最慢的一次，要尽量保证Get阶段稳定
* Aep保证8字节的原子性写

    * Aep以及内存保证按8字节对齐后小于等于8字节的写入是原子性的

### 3.3 初版方案

#### 3.3.1 架构

* Aep hash索引：用来对Key进行分桶，以及满足持久化要求
* hash索引缓存：对Aep hash索引进行缓存，对内存是快于读Aep
* 缓存：Get阶段对读进行缓存

#### 3.3.2 数据结构

##### 1 索引和KV的数据结构

* Aep有64G，数据块按照16字节对齐之后，只要4个字节就能表示数据块地址
* 同一个hash桶内kv数据之间使用链表存储，insert使用头插法，insert的时间复杂度就是O(1)
* 由于key是均匀分布的，所以使用的是key的前8个字节的低28位作为hash值
* val_ptr、block_size和val_len一起是8字节并且起始地址是按照8字节对齐，可以保证val更新符合持久化要求

Aep的分配策略是按照4M块进行分配的，每个线程写完一个4M之后再申请下一个4M块，如下图所示。

这样带来的好处有：
* 减小锁的粒度：无需每次写都去获取锁
* 减少随机写：线程内基本顺序写，线程间相对随机写

##### 2 内存回收池数据结构

数据块回收：找数据块大小对应的栈，入栈即可，时间复杂度为O(1)

数据块复用：从数据块大小对应的栈开始按照一定步长找不为空栈，然后出栈即可，时间复杂度为O(1)

##### 3 缓存数据结构

* 使用覆盖的方式解决hash冲突
* 读缓存需要先加分段锁，保证缓存并发读写安全

#### 3.3.3 set操作

set操作流程图如下所示。

数据写入大于8字节是非原子操作，索引写入是8字节，aep可以保证原子写，所以需要先写数据再写索引

**1、置换更新**

置换更新val数据结构如下图所示。

每个线程会预留一块aep置换区数据块用于val置换更新(更新后val小于等于更新前val数据块则可以使用置换更新)

置换更新示例图。

置换更新接住了大概2/3的update操作。

**2、key插入操作**

key insert写入数据后需要更新文件索引以及文件索引缓存，如下图所示。

**3、key更新操作**

key更新val写入数据后，需要同时更新val_ptr、block_size和val_len，具体如下图所示。

#### 3.3.4 get操作

get操作流程图如下所示。

#### 3.3.5 方案效果

* 内存使用量：2633M
* hash索引缓存：2G
* 缓存：522M
* 回收池：64M
* 缓存命中率：98%
* hash效果
    * 第1层：67.2%
    * 第2层：24.6%
    * 第3层：6.6%
    * 第4层以上：1.6%

* 每次set需要写2次Aep：1次写Aep数据，1次写Aep索引 
* 最优耗时：55s（set阶段41s，get阶段14s）

### 3.4 优化版方案

#### 3.4.1 架构

和初版方案不同点在于索引和KV数据存储

#### 3.4.2 索引和KV的数据结构

* Key的数量有220M，每个Key16字节，总共有不到3.5G的Key，内存有8个G，所以可以对所有Key进行缓存
* 内存key索引各数据项按照访问先后顺序组织，充分利用缓存行
* key insert的version从1开始，key update的version会等于原来key的version+1，版本越大，表明数据越新
* 数据块每4个字节累加作为crc值

内存key索引分配策略和Aep分配策略类似，如下图所示。

每个线增加Key索引之前获取长度为1024*128的空间，写完之后再申请下一个1024*128，减小锁的粒度。

##### 1 索引重建流程

程序初始化的时候，根据Aep分配策略以及存储KV的数据结构，重建索引，具体流程如下图所示。

> 在索引重建过程中，把crc不相等以及版本号低的数据块进行回收

#### 3.4.3 优化效果

* 每次set只要写1次Aep，没有任何读Aep操作
* 每次get读，没有命中缓存才有1次读Aep操作（key在内存都有，只需要读1次val）
* 内存使用量：7958M
    * hash索引：1G
    * key索引：6.2G
    * 缓存：522M
    * 回收池：64M
* 缓存命中率和hash效果同初版方案
* 最优耗时：35.8s（set阶段27.1s，get阶段8.7s）
* 最终排名：第3名

#### 3.4.5 其他优化

* 初始化时对Aep预写：耗时节省了7.8s左右
* get操作val结果对象复用：耗时节省了2s左右
* 全局回收池换成thread local回收池：耗时节省了1s左右
* mutex互斥锁换成spin自旋锁：耗时节省了100多ms
* 一维数组模拟二维数组、减少对象产生
* 控制Get阶段Set的稳定性：Get阶段的Set是有热点Key更新的特征并且基本全部满足置换更新的条件，所以可以让每次Set更新前后的偏移差保证在一定范围内，从而大大减少更新落盘的随机性

## 4 总结

顺序读写Aep优于随机读写，以及尽量减少IOPS

无锁或者尽可能较少锁冲突：使用thread_local变量；加锁优先使用分段锁、自旋锁、一次分配多个的方式来减小锁粒度和冲突

利用缓存行，避免伪共享等

充分利用内存来减少Aep操作

