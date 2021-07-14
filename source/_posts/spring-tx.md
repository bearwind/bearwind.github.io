title: spring多线程事务小记
tags:
  - spring
  - transaction
categories: 谈技论术
date: 2021/03/25 16:50

---
### 问题背景
> 定时批处理任务，需要执行大量客户账单，为提升效率，减少DB占用时间，从而采用多线程方式执行任务。主线程任务执行与子线程任务需要保证在一个事务中，完成原子性任务操作。但实际情况是，即使写两个独立service（由于AOP代理，方法内部调用不走代理），并且分别加上``@Transactional``，只能保证各自的线程事务生效，无法做到全局事务。

### 问题探讨
> 通过源码可看到：***code copied from "org.springframework.transaction.interceptor.TransactionAspectSupport"***

```
/**
 * Holder to support the {@code currentTransactionStatus()} method,
 * and to support communication between different cooperating advices
 * (e.g. before and after advice) if the aspect involves more than a
 * single method (as will be the case for around advice).
 */
private static final ThreadLocal<TransactionInfo> transactionInfoHolder =
		new NamedThreadLocal<TransactionInfo>("Current aspect-driven transaction");


/**
 * Opaque object used to hold Transaction information. Subclasses
 * must pass it back to methods on this class, but not see its internals.
 */
protected final class TransactionInfo {

	private final PlatformTransactionManager transactionManager;

	private final TransactionAttribute transactionAttribute;

	private final String joinpointIdentification;

	private TransactionStatus transactionStatus;

	private TransactionInfo oldTransactionInfo;

	public TransactionInfo(PlatformTransactionManager transactionManager,
			TransactionAttribute transactionAttribute, String joinpointIdentification) {

		this.transactionManager = transactionManager;
		this.transactionAttribute = transactionAttribute;
		this.joinpointIdentification = joinpointIdentification;
	}
```

事务信息``TransactionInfo``，在线程间是通过``ThreadLocal``进行隔离，每个线程存储自己的副本，因而事务在线程间不共享，所以即使加了``@Transactional``，也不会使线程间共享一个事务。即：
serviceA主线程方法加``@Transactional``，serviceB子线程方法加``@Transactional``，子线程异常会回滚自己，但主线程无法感知，所以非异常不会回滚，导致主线程和子线程事务不一致。

### 问题解决
#### 手动回滚
通过thread的``setUncaughtExceptionHandler``，设置未捕获异常处理，然后通过标记，在主线程进行手动回滚

```
// 主线程方法
// 用于标记子线程异常
private volatile boolean flag = false;
// ...省略
@Transactional
    public void first () {
        StockIndex s = new StockIndex();
        s.setStockId("123");
        s.setStockType(2);
        s.setAccessDate(new Date());
        sim.insertSelective(s);

        Thread t = new Thread(() -> {
            testTx2.second();
        });

        t.setUncaughtExceptionHandler((th, e) -> {
            System.out.println("uncaunght");
            flag = true;
        });
        t.start();
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //throw new RuntimeException("stop!");

        if (flag) {
            TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
        }
    }
    // 子线程方法
    @Transactional
    public void second () {
        StockIndex s = new StockIndex();
        s.setStockId("1234");
        s.setStockType(1);
        s.setAccessDate(new Date());
        sim.insertSelective(s);
        throw new RuntimeException("!!!");
    }
    // 调用
    @Test
    public void testTx () {
        testTx.first();
    }
```

### 关于spring事务失效场景
+ service里方法内部调用，由于内部调用不走代理；
+ 多线程事务，各自事务相互独立；
+ ``@Transactional``加载了接口方法上，由于该注解不支持接口级别的继承，源于jdk元注解：
> ***copied from java.lang.annotation.Inherited***
 Note also that this meta-annotation only causes annotations to be inherited
 from superclasses; annotations on implemented interfaces have no effect.
+ 加注解的方法，非public，导致代理无法生效
+ ...
