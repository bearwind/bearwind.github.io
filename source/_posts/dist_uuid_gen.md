title: 分布式环境下的UUID生成策略
tags:
  - java
  - middle-ware
  - distribution
categories: 谈技论术
date: 2018/08/25 10:12


---

### 0 **前言**
> UUID(Universally Unique Identifier), 通用唯一识别码， 其产生的目的就是为了在分布式的大环境下让每一个独立的有意义的元素都有其唯一的身份识别码，如同我们的身份证号一样。

### 1 **抛砖**
#### 1. java原生UUID生成方式
+ **示例**
```java
String uuid = UUID.randomUUID().toString();
System.out.println(uuid);
//output: 52f3a858-cd2d-4994-b4f1-9a06839deb45
```
+ **缺点**
1. UUID随机生成长度为36位的字符串，太长，占用空间较大，就算去掉``-``，还是很长
2. 丑陋，反人类的识别度
3. 无序，若排序几无可能
4. 索引效率特别低，入库性能差

#### 2. mysql数据库自增ID
+ **示例**
创建名为uuid的表，结构如下

|id|field|
|:-:|:-:|
|1|a|
|PK|char(1)|
然后每次获取id:
```sql
REPLACE INTO uuid (field) VALUES ('a');
select LAST_INSERT_ID();
```
当然，为了提高性能，体现分布式的优势，我们可以采用分库的方式，使用proxy请求不同的分库，如图
![proxy分库生成uuid](http://5b0988e595225.cdn.sohucs.com/images/20180518/2ac0cbcc884b487391f412a08020ab7d.png)
这样一来，DB1生成的ID是1,4,7,10,13....，DB2生成的ID是2,5,8,11,14.....

+ **缺点**
ID生成严重依赖mysql，每次生成都需要请求mysql，在高并发环境下，性能不是很好，如果数据库挂掉，会影响业务

### 2 **引玉**
#### 1. Twitter的SnowFlake 雪花算法
+ **概述**
![snowflake data structure](http://5b0988e595225.cdn.sohucs.com/images/20180518/1af4eb1ec81d4501a757cfdc9ada72d1.jpeg)

``1位``，不用。二进制中最高位为1的都是负数，但是我们生成的id一般都使用整数，所以这个最高位固定是0用来记录时间戳（毫秒）。

``41位``，用来记录时间戳（毫秒）。
- [x] 可以表示2^41个数，
- [x] 如果只用来表示正整数（计算机中正数包含0），可以表示的数值范围是：0 至 2^41−1，减1是因为可表示的数值范围是从0开始算的，而不是1。
- [x] 也就是说41位可以表示2^41−1个毫秒的值，转化成单位年则是(2^41−1)/(1000∗60∗60∗24∗365)=69年

``10位``，用来记录工作机器id。
- [x] 可以部署在2^10=1024个节点，包括5位``datacenterId``和5位``workerId``
- [x] 5位（bit）可以表示的最大正整数是2^5−1=31，即可以用0、1、2、3、....31这32个数字，来表示不同的datecenterId或workerId

``12位``，序列号，用来记录同毫秒内产生的不同id。
- [x] 12位（bit）可以表示的最大正整数是2^12−1=4095，即可以用0、1、2、3、....4095这4096个数字，来表示同一机器同一时间戳（毫秒)内产生的4096个ID序号

+ **SHOW THE CODE**
```
// 简单粗暴解决时钟回拨：
public synchronized long nextId () {
	long timestamp = timeGen();

	//如果当前时间小于上一次ID生成的时间戳，说明系统时钟回拨过，这个时候应当抛出异常，或阻塞时间差的时间
	int times = BACKWARDS_MAX_COUNT;
	if (timestamp < lastTimestamp) {
	    for (long diff = lastTimestamp - timestamp;
		 diff <= 5 && diff > 0 && times <= BACKWARDS_MAX_COUNT;
		 timestamp = timeGen(), diff = lastTimestamp - timestamp, times--) {
		try {
		    Thread.sleep(diff);
		} catch (InterruptedException e) {
		    // ignore
		}

	    }
	    throw new RuntimeException(
		    String.format("Clock moved backwards too many times. Refusing to generate id for %d milliseconds",
		            lastTimestamp - timestamp));
	}
```
+ **解惑**
```
// 时间戳差值导致ID位数变化
double v = Math.pow(10, 17);
long val = new Double(v).longValue();
long year =1000 * 60 * 60 * 24 * 365L;
System.out.println((double)(val >> 22) / year);
```
#### 2. Redis生成自增
+ **概述**
使用redis的原子性自增命令``incr``（每次自增1）和``incrby``（每次自增step）
+ **优点**
 1. 有序、可读性强，可以用时间戳+自增id，或者每天一个新自增key``set ${today}uuid 0``，每天从0开始
 2. 性能不错，高可用
+ **缺点**
 1. 强依赖redis，每次获取id必须请求redis
 2. redis宕机，会对影响业务，当然可以采用多节点，每个主起始id和step不同等优化

#### 3. Zookeeper维护版本号
+ **概述**
使用zookeeper的数据版本号来作为自增，由于zk属于CP型分布式系统，因而高并发下性能会差一些，而且有些时候需要设置分布式锁
+ **SHOW THE CODE**
> 利用线程池并发设置和获取version（同一条操作）,用``set add``添加去重所有version，最终version数量符合预期，无重复

```
public static void main (String[] args) throws InterruptedException {
        // 基于curator封装的zk操作工具类
        ZkOperator o = new ZkOperator("warwick01:2181,warwick02:2181,warwick03:2181", 120000, 200000);
        o.connect();
        o.delete("/seq");
        o.create("/seq");
        int cpus = Runtime.getRuntime().availableProcessors();
        ExecutorService es = new ThreadPoolExecutor(cpus, 
                                                    2 * cpus + 1, 
                                                    1, TimeUnit.SECONDS,
                                                    new LinkedBlockingQueue<>(91));
        Set<Integer> set = new ConcurrentHashSet<>();
        CountDownLatch latch = new CountDownLatch(100);
        for (int i = 0; i < 100; i++) {
            es.submit(() -> {
                try {
                    Stat a = o.cf.setData().withVersion(-1).forPath("/seq", "1".getBytes());
                    set.add(a.getVersion());
                    System.out.println(a.getVersion());
                    latch.countDown();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });
        }
        latch.await();
        System.out.println("size: " + set.size());
        es.shutdown();
    }
```

#### 4. mysql号段
+ **概述**
![mysql号段](https://gitee.com/bearwind/image_host/raw/master/2021-06/mysql_uuid_plus.png)
我们看上图，有张ID规则表：

1. id表示为主键，无业务含义。
2. biz_tag为了表示业务，因为整体系统中会有很多业务需要生成ID，这样可以共用一张表维护
3. max_id表示现在整体系统中已经分配的最大ID
4. desc描述
5. update_time表示每次取的ID时间

我们再来看看整体流程：

1. 【用户服务】在注册一个用户时，需要一个用户ID；会请求【生成ID服务(是独立的应用)】的接口
2. 【生成ID服务】会去查询数据库，找到user_tag的id，现在的max_id为0，step=1000
3. 【生成ID服务】把max_id和step返回给【用户服务】；并且把max_id更新为max_id = max_id + step，即更新为1000
4. 【用户服务】获得max_id=0，step=1000；
5. 【用户服务】可以用ID=【max_id + 1，max_id+step】区间的ID，即为【1，1000】
6. 【用户服务】会把这个区间保存到jvm中
7. 【用户服务】需要用到ID的时候，在区间【1，1000】中依次获取id，可采用AtomicLong中的getAndIncrement方法。
8. 如果把区间的值用完了，再去请求【生产ID服务】接口，获取到max_id为1000，即可以用【max_id + 1，max_id+step】区间的ID，即为【1001，2000】

>  这个方案就非常完美的解决了数据库自增的问题，而且可以自行定义max_id的起点，和step步长，非常方便扩容。
而且也解决了数据库压力的问题，因为在一段区间内，是在jvm内存中获取的，而不需要每次请求数据库。即使数据库宕机了，系统也不受影响，ID还能维持一段时间。

#### 5. mysql号段+双Buffer
+ **概述**
为解决高并发，请求将号段用完后，阻塞竞争mysql的问题
![mysql双Buffer](https://gitee.com/bearwind/image_host/raw/master/2021-06/mysql_2_buffers.png)
1. 当前获取ID在buffer1中，每次获取ID在buffer1中获取
2. 当buffer1中的Id已经使用到了100，也就是达到区间的10%
3. 达到了10%（根据业务设定阈值），先判断buffer2中有没有去获取过，如果没有就立即发起请求获取ID线程，此线程把获取到的ID，设置到buffer2中。
4. 如果buffer1用完了，会自动切换到buffer2
5. buffer2用到10%了，也会启动线程再次获取，设置到buffer1中
6. 依次往返

> + 双buffer的方案，满足业务场景所用ID，而且都是在jvm内存中获得的，不需要从DB中获取，效率飙升；
+ 允许数据库宕机时间更长，因为会有一个线程作为”看门狗“，两个buffer来回切换使用，解决了突发阻塞的问题。
