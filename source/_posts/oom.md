title: 记一次生产OOM事故排查
tags:
  - java
  - jvm
categories: 研海搏浪
date: 2021/01/26 13:50

---

### 事故背景
> 某异步建模项目接口产品，客户入参时间段（一年之内），mid/pid等，查询商户日增指标及日汇总流水，并将数据汇总合并后输入客户模型计算评分。

### 事故出现
生产监控报警，服务不可用，队列大量堆积。查看生产日志，消费者服务报OOM：

```
2021-01-25 11:45:32.654 [pool-3-thread-100] ERROR org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer[1524]: Consumer thread error, thread abort.
java.lang.OutOfMemoryError: Java heap space
        at java.lang.AbstractStringBuilder.<init>(AbstractStringBuilder.java:68)
        at java.lang.StringBuilder.<init>(StringBuilder.java:101)
        at com.fasterxml.jackson.core.util.TextBuffer.contentsAsString(TextBuffer.java:346)
        at com.fasterxml.jackson.core.io.SegmentedStringWriter.getAndClear(SegmentedStringWriter.java:83)
        at com.fasterxml.jackson.databind.ObjectWriter.writeValueAsString(ObjectWriter.java:1037)
        at com.xx.odin.util.JSONUtils.toJson(JSONUtils.java:52)
        at com.xxx.odin.util.JSONUtils.toJson(JSONUtils.java:35)
        at com.xxx.lzual.trans.service.impl.LzualServiceImpl.merchantBusinessFigure(LzualServiceImpl.java:256)
        at com.alibaba.dubbo.common.bytecode.Wrapper1.invokeMethod(Wrapper1.java)
        ...
```
### 问题排查
+ 排查相关代码，未发现明显问题。
然后执行命令，导出堆内存信息dump文件：

```
jmap -dump:format=b,file=dump <pid>
```
+ 然后将5个多G的dump文件导入jdk自带的``jvisualvm``可视化工具进行分析：
![oom01](https://gitee.com/bearwind/image_host/raw/master/2021-01/oom01.png)
+ 查看具体的异常线程信息：
![oom02](https://gitee.com/bearwind/image_host/raw/master/2021-01/oom02.png)

可以看出：给对象``char[]``分配内存时报了OOM错误，具体操作是在用``JsonUtils``转换对象打印日志，**jackson底层进行了``Arrays.copyOfRange()``操作**，导致堆内存占用进一步加大。
+ 查看类图：
![oom03](https://gitee.com/bearwind/image_host/raw/master/2021-01/oom03.png)
存在74055个``char[]``对象，总大小就5.18G，线上服务使用的是2C8G的服务器，启动服务占用加上杂七杂八的，这分分钟就给你溢出了。
+ 根据下图对象中具体数据，结合类图，内存溢出问题最终定位在``DayData``这个dto对象。
![oom04](https://gitee.com/bearwind/image_host/raw/master/2021-01/oom04.png)
+ 再一次排查代码，问题定位在如下代码块：

```
for (String mid : midSurvivors) {
    // 日汇总流水
    List<DayData> daySum = hbaseService.getMidDaySum(mid, start, end);
    // 日增指标
    List<DayData> dayUpIndex = hbaseService.getMidDayUpIndex(mid, start, end);
    // 合并汇总
    List<DayData> entire = combineDayData(daySum, dayUpIndex);
    ...
}
```
通过多个mid查询hbase获取两类数据，由于客户传入的时间段间隔几乎都接近一年（360天），而且查询的mid都是活跃商户，每日日汇总流水明细``List<DayData>``非常大，这样导致数据量巨大，这一部分数据大概1.73G：
![oom05](https://gitee.com/bearwind/image_host/raw/master/2021-01/oom05.png)
### 问题解决
发现问题就是对代码进行优化。
1. 将使用后的大对象显示置null，是jvm更容易GC掉；
2. 最开始想显示使用``System.gc()``，但是通过查询资料显示，该方法会出发FullGC，不合适弃用；
3. 去除每个对象中的多余无用字段，从之前20多个到最后保留10多个；
4. 限制客户入参，缩短调用时间段；
5. 去除无用日志打印、json转换。
### 扩展阅读
[Java大对象优化](https://blog.csdn.net/vincent_wen0766/article/details/111572519)

### 模拟OOM
设置JVM参数：``-Xms10M -Xmx20M``

```
public static void main (String[] args) {
    new Thread(() -> {
        List<SomeObj> list = new ArrayList<>();
        for (;;) {
            SomeObj obj = new SomeObj();
            obj.setName("test");
            list.add(obj);
        }
    }).start();

    new Thread(() -> {
       for (;;) {}
    }).start();
}
```
异常出现

```
Exception in thread "Thread-0" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3210)
	at java.util.Arrays.copyOf(Arrays.java:3181)
	at java.util.ArrayList.grow(ArrayList.java:261)
	at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:235)
	at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:227)
	at java.util.ArrayList.add(ArrayList.java:458)
	at com.novawind.MemoryTest.lambda$main$0(MemoryTest.java:18)
	at com.novawind.MemoryTest$$Lambda$1/401625763.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:745)
```
``Arrays.copyOf``处报错，可见list扩容时，由于复制操作，会占用更多内存，导致OOM异常。
