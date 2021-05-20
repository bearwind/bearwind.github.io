title: 项目重构回顾：使用模版、策略、桥接设计模式
tags:
  - java
  - design-pattern
categories: 谈技论术
date: 2021/03/12 10:25

---
### 重构背景
> 公司某模型项目，按产品接口特点，分为四种属性：产品特性维度**Dimension**、调用方式**CallerType**、模型部署类型**DeployType**，模型编号**ModelNumber**，前两个属性决定了唯一产品接口，后两个属性决定具体产品下模型具体操作。
由于该项目设计初期的不完善、快速的需求迭代变更、开发人员信息同步不及时，导致这个项目俨然成为了一座***屎山***：
大量的冗余代码、过度的if-else嵌套等等问题，使得对接新产品、维护旧产品变得异常困难！所以重构势在必行。

### 设计模式
#### 模版模式
+ 应用点：调用不同模型前，需要处理不同的模型入参，但通过模型DeployType选择模型具体的Executor等是共有功能。因而将共功能形成模版方法(``handle()``)，将处理入参(``deal()``)抽离出来暴露给使用者进行自定义。

```
/**
 * 执行者处理器模版接口
 *
 * @author Jeremy Xiong<br>
 * 2021-03-09 10:55
 */
public interface ExecutorHandlerTemplate {
    Logger log = LoggerFactory.getLogger(ExecutorHandlerTemplate.class);

    /**
     * 执行者处理器模版处理方法
     *
     * @param request
     * @return
     */
    default ModelUnionResult handle (ModelUnionRequest request) {
        // 相关业务代码 pre
        ExecutorInput input = deal(request); // 需要自定义实现的方法部分
        // 相关业务代码 post
        return result;
    }

    /**
     * 不同dimension处理入参的方法
     *
     * @param request
     * @return
     */
    ExecutorInput deal (ModelUnionRequest request);

}

```

这样使用者只需要关心自己那一部分独特的入参方法就行了：

+ 具体实现

```
/**
 * 卡维度实现
 *
 * @author Jeremy Xiong<br>
 * 2021-03-09 11:47
 */
@Handler(Dimension.CARD)
public class CardExecutorHandlerImpl implements ExecutorHandlerTemplate {

    @Override
    public ExecutorInput deal (ModelUnionRequest request) {
        // 具体参数入参方法代码
        return input;
    }
}
```
通过模版方法，将各种Dimension的冗余代码全部删除，太舒服了...
#### 策略模式
+ 应用点：部署类型DeployType（Docker、DecisionEngine、http、pmml等）具体的executor策略接口

```
/**
 * 模型部署类型执行器策略
 *
 * @author Jeremy Xiong<br>
 * 2021-03-04 20:54
 */
public interface IExecutorStrategy {

    ModelUnionResult execute (ExecutorInput input);
}
```
+ 具体实现示例
```
/**
 * @author Jeremy Xiong<br>
 * 2021-03-12 09:57
 */
@Executor(DeployType.DOCKER)
@Slf4j
public class DockerExecutor implements IExecutorStrategy {
    @Autowired
    private DubboService dubboService;

    @Override
    public ModelUnionResult execute (ExecutorInput input) {
        // 业务代码
        IDockerService docker = dubboService.get(IDockerService.class);
        ModelUnionResult result = docker.execute(input);
        // 业务代码
        return result;
    }
```
+ 调用时，通过spring的特性来注册策略
```
@Autowired
private ApplicationContext ctx;
// 注册
@Bean(Constants.EXECUTOR_STRATEGY)
    public Map<DeployType, IExecutorStrategy> registerExecutor () {
        Map<String, IExecutorStrategy> map = ctx.getBeansOfType(IExecutorStrategy.class);
        Map<DeployType, IExecutorStrategy> strategyMap = new HashMap<>();
        for (IExecutorStrategy ie : map.values()) {
            strategyMap.put(key(ie), ie);
        }
        return strategyMap;
    }
```
+ 获取策略
```
// default方法或static方法中
Map strategyMap = ApplicationContextUtil.getBean(Constants.EXECUTOR_STRATEGY, Map.class);
IExecutorStrategy executor = (IExecutorStrategy) strategyMap.get(deployType);
result = executor.execute(input);
// 注入获取
@Autowired
@Qualifier(Constants.EXECUTOR_STRATEGY)
private Map<DeployType, IExecutorStrategy> strategyMap;
// ...
IExecutorStrategy executor = strategyMap.get(deployType);
result = executor.execute(input);
```
通过策略模式，彻底与繁琐的if-else说拜拜！
***当然，在调用模型时，需要根据具体的模型编号调用具体模型，也同样使用了策略模式***

#### 混合应用
+ 应用点：1. 调用类型CallerType，分为同步（SYNC）和异步（ASYNC）,抽象一个调用策略类，然后各自具体实现：

```
/**
 * 调用策略
 * <p>使用前请注册：{@link CallerStrategyFactory}
 * @author Jeremy Xiong<br>
 * 2021-03-09 15:36
 */
public abstract class AbstractCallerStrategy {
    protected final PreHandler preHandler;
    protected ExecutorHandlerTemplate template;


    public AbstractCallerStrategy (PreHandler preHandler) {
        this.preHandler = preHandler;
    }
    public AbstractCallerStrategy (PreHandler preHandler, ExecutorHandlerTemplate template) {
        this.preHandler = preHandler;
        this.template = template;
    }
    public abstract ModelUnionResponse call (ModelUnionRequest request);
```
其实可能有人已经注意到，该类既为调用策略CallerType的抽象类，又是``Prehandler``和``ExecutorHandlerTemplate``的桥接类。
也就是同时运用了策略模式和桥接模式（见下文）。

+ 同步具体实现示例，使用**策略模式**和**模版模式**

```
/**
 * 同步调用策略
 *
 * @author Jeremy Xiong<br>
 * 2021-03-09 20:45
 */
@Slf4j
public class SyncCallerStrategy extends AbstractCallerStrategy {


    public SyncCallerStrategy (PreHandler preHandler, ExecutorHandlerTemplate template) {
        super(preHandler, template);
    }

    @Override
    public ModelUnionResponse call (ModelUnionRequest request) {
        // 业务代码
        ModelUnionResponse response = preHandler.pre(request); // 模版设计模式，自定义预处理器的pre方法
        // 业务代码
        return response;
    }
}
```
+ 预处理器

```
/**
 * 预处理器
 *
 * @author Jeremy Xiong<br>
 * 2021-03-09 20:38
 */
public interface PreHandler {

    /**
     * 预处理：各种行为拼装request
     *
     * @param request
     */
    ModelUnionResponse pre (ModelUnionRequest request);

```
+ 通过构造函数，spring bean注册，运用**桥接模式**将预处理器（``PreHandler``）的具体实现和执行器模版（``ExecutorHandlerTemplate``）的具体实现桥接起来完成调用策略的实现


```
/**
 * 调用策略注册工厂
 *
 * @author Jeremy Xiong<br>
 * 2021-03-10 10:20
 */

@Configuration
public class CallerStrategyFactory {

    @Bean(Caller.CARD + Caller.SYNC)
    public AbstractCallerStrategy cardSyncCallerHandler (CardPreHandlerImpl h, CardExecutorHandlerImpl e) {
        return new SyncCallerStrategy(h, e);
    }

    @Bean(Caller.CARD + Caller.ASYNC)
    public AbstractCallerStrategy cardAsyncCallerHandler (CardPreHandlerImpl h) {
        return new AsyncCallerStrategy(h);
    }

    @Bean(Caller.CARDS + Caller.SYNC)
    public AbstractCallerStrategy cardsSyncCallerHandler (CardsPreHandlerImpl h, CardsExecutorHandlerImpl e) {
        return new SyncCallerStrategy(h, e);
    }

    @Bean(Caller.CARDS + Caller.ASYNC)
    public AbstractCallerStrategy cardsAsyncCallerHandler (CardsPreHandlerImpl h) {
        return new AsyncCallerStrategy(h);
    }

    @Bean(Caller.PERSON + Caller.ASYNC)
    public AbstractCallerStrategy personAsyncCallerHandler (PersonPreHandlerImpl h){
        return new AsyncCallerStrategy(h);
    }

    @Bean(Caller.PERSON + Caller.SYNC)
    public AbstractCallerStrategy personSyncCallerHandler (PersonPreHandlerImpl h, PersonExecutorHandlerImpl e){
        return new SyncCallerStrategy(h, e);
    }

    @Bean(Caller.MERCHANTS + Caller.ASYNC)
    public AbstractCallerStrategy merchantsSyncCallerHandler (MerchantNameHandlerImpl h){
        return new AsyncCallerStrategy(h);
    }
```

### 总结
1. 模版模式：抽出公共业务代码，使用者只关注某一块特有的实现
2. 策略模式：将同一类型策略抽象出来，通过特有的标记获取某一特定策略，实现对扩展开放，对修改关闭
3. 桥接模式：当多个模块的抽象类或实现需要最终组合成某一完整的功能时，使用桥接模式进行拼装桥接

### 重构的简明UML类图
![UML类图](https://gitee.com/bearwind/image_host/raw/master/2021-03/rebuild-UML.png)






