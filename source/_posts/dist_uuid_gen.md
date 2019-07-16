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
- [x] 可以表示2^41−1个数字，
- [x] 如果只用来表示正整数（计算机中正数包含0），可以表示的数值范围是：0 至 241−1，减1是因为可表示的数值范围是从0开始算的，而不是1。
- [x] 也就是说41位可以表示241−1个毫秒的值，转化成单位年则是(241−1)/(1000∗60∗60∗24∗365)=69年
10位，用来记录工作机器id。
可以部署在210=1024个节点，包括5位datacenterId和5位workerId
5位（bit）可以表示的最大正整数是2^5−1=31$，即可以用0、1、2、3、....31这32个数字，来表示不同的datecenterId或workerId
12位，序列号，用来记录同毫秒内产生的不同id。
12位（bit）可以表示的最大正整数是2^12−1=4095，即可以用0、1、2、3、....4094这4095个数字，来表示同一机器同一时间截（毫秒)内产生的4095个ID序号

``10位``，用来记录工作机器id。
- [x] 可以部署在2^10=1024个节点，包括5位``datacenterId``和5位``workerId``
- [x] 5位（bit）可以表示的最大正整数是$2^5−1=31$，即可以用0、1、2、3、....31这32个数字，来表示不同的datecenterId或workerId

``12位``，序列号，用来记录同毫秒内产生的不同id。
- [x] 12位（bit）可以表示的最大正整数是$2^{12}−1=4095$，即可以用0、1、2、3、....4094这4095个数字，来表示同一机器同一时间戳（毫秒)内产生的4095个ID序号

+ **SHOW THE CODE**
...
+ **解惑**
```
// 时间戳差值导致ID位数变化
double v = Math.pow(10, 17);
long val = new Double(v).longValue();
long year =1000 * 60 * 60 * 24 * 365L;
System.out.println((double)(val >> 22) / year);
```

#### 2. Redis生成自增
