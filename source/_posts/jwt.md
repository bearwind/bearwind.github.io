title: 记一次JWT鉴权问题解决
tags:
  - auth
  - middle-ware
categories: 谈技论术
date: 2019/05/12 14:25

---
### 问题背景
 
1. 前后端分离项目，采用统一api网关，使用JWT + redis作为鉴权方式；
2. 用户登录验证通过，颁发JWT，设置JWT过期时间为2h，redis缓存该用户JWT：``set(user:${id}, "Bearer ...")``，并设置该key的TTL为7天；将该有效token通过header返回前端
3. 鉴权过程：
   + 在header里获取token（token-A）并解析获取userId等信息从而获取redis key；
   + 查询redis获取当前用户token：
        - 返回为空（redis过期或未登录等），则该token无效，直接拦截；
        - 返回非空（token-B），则校验token-A与token-B是否相等，不相等则说明token已被更换，直接拦截
        - 校验token自身过期时间，若已过期（expire > 2h），由于redis存在该token，但在允许刷新时间内（保持登录时间），系统重新颁发新token，用户无感知。

### 问题出现
> 某功能页面，渲染的时候会同时调用多个后端接口，一般情况下都是正常的，但偶尔出现渲染一部分数据后会被后端拦截提示未授权，然后跳转登录页。
经分析：前端同时调用A、B接口，header中的token为同一个。调用A接口时，token自身正好过期，然后更换为新token，导致调用B接口时由于新旧token不一致被拦截

### 问题解决
1. 当更换新token时，在redis存放一个TTL非常短的（10s或更短）缓存标记，以新旧token为key-value存储的token-flag，``set(token-old, token-new)``；
2. 所有接口调用，先校验是否过期，过期则查询1中的token-flag是否存在，存在说明刚刚更换新token，直接放行，无则继续走之前的鉴权过程；
3. 大致过程：
 1) 请求A -> token-old过期 -> 颁发新token token-new -> 存储token-flag -> 将token-new放入header返回
 2) 请求B -> token-old过期 -> 查询token-flag存在 -> 放行（说明在更换新token的保护期内），并将token-flag中的value值token-new放入header返回
至此token更换已同步给前端

> 你以为这样就彻底解决了吗？那就大错特错了！

### 方案优化
当token过期时，我们会校验token-flag是否存在，但前端有很多页面都是分模块展示数据，对于这些没有前后调用关系的接口，前端会同时并发调用然后渲染各个模块的数据，提高加载速度、提升用户体验。
那么问题就来了：**并发**，老问题户了。
当请求并发进来时，A、B请求都发现token-old过期，然后都同时更换新token，存储token-flag，但set操作会覆盖之前的新token出现问题
#### 解决
token过期，使用``SETNX``设置token-flag，保证只会生成唯一的新token

### 功能迭代
> 业务说需要支持一个账号多个点登录（登录电脑数n >= 2）

**满足她！**
变更token存储的redis数据结构，string -> list，使用``RPUSHX``存储多个登录端的token，``LPOP``大于登录数n的旧token，存放新登录端的token，保证list长度小于n.
