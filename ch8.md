# 第8章 网关：Zuul 过滤器链与路由边界

**一句话**：网关把鉴权、路由、限流、日志收成一条过滤器链，代价是它自己成了必须被冗余的单点。

**为什么需要它**：第 12 个 Provider 上线时，运维照例又配了一遍 token 校验、跨域白名单、访问日志和限流阈值；四份逻辑散在 12 个仓库里，改一次校验规则要发 12 次版。更要命的是客户端自己维护实例列表并做负载均衡，一个实例已经下线，请求还在往它发，报错散落在每个调用方。把这些横切关注点（cross-cutting concern）收进一个入口后，问题并没有消失，只是换了形态：Nginx 把 80 端口的流量全部 proxy_pass 到唯一的 Zuul upstream，一旦这个进程挂了，整站所有服务同时不可达；而某个 pre 过滤器里顺手写了订单状态判断之后，每条业务线改需求都要重启网关。这一章要回答的是：外部网关与内部网关各自该管哪一段，四类过滤器按什么契约串成一条链，路由规则怎么和服务发现拼起来，以及为什么在网关里写业务逻辑、把网关当单点，是两种必须提前挡住的设计。

**核心结构**：

| 概念 | 作用 |
|---|---|
| 外部网关 Nginx / OpenResty | 扛连接、TLS、静态资源、keepalive，反向代理（reverse proxy）到 Zuul |
| 内部网关 Zuul（@EnableZuulProxy） | 服务级路由 + 业务横切，自身也是注册到 Eureka 的微服务 |
| ZuulFilter | 自定义过滤器抽象基类，四个契约方法 |
| filterType / filterOrder | 阶段类型；同类型内按值升序，越小越先执行 |
| shouldFilter / run | 是否生效的开关；真正承载拦截、改请求、改响应的逻辑 |
| RequestContext | 每请求唯一，过滤器之间不直接通信、只共享这个上下文 |
| RibbonRoutingFilter | route 阶段结合 Ribbon 选实例、Hystrix 做隔离并发起调用 |
| serviceId → 实例列表 | 路由规则结合服务发现（service discovery）做动态转发 |
| sensitiveHeaders / strip-prefix | 屏蔽敏感请求头；决定是否剥掉路由路径前缀 |
| error 过滤器 | 任一阶段抛异常后统一接管，可直接生成最终响应 |

**关键设计**：

```text
  客户端 -> Nginx(外部网关) -> Zuul(内部网关) -> Provider 实例
                                 |
  pre 阶段   [同类按 filterOrder 升序]  shouldFilter? -> run()
     |  鉴权 / 黑名单 / 灰度标 / 追加 USER-ID 头
     |  ctx.setSendZuulResponse(false) -> 后续 route 过滤器全部跳过
     v
  route 阶段  RibbonRoutingFilter：serviceId -> 实例列表 -> 选定并调用
     |        （Ribbon 负载均衡 + Hystrix 线程池隔离 / 超时）
     v
  post 阶段  加响应头 / 收集指标与统计 / 把结果回写客户端
     |
  任意阶段抛异常 ----> error 阶段（统一处理，可返回最终结果）

  每请求一个 RequestContext：过滤器间不互相调用，只读写这个共享上下文
```

```yaml
# 示意：路由规则，非原书完整配置
zuul:
  ribbonIsolationStrategy: THREAD          # 命令调到独立线程池，隔离由此而来
  host:
    connect-timeout-millis: 600000        # 建连超时；调大不等于下游更快
    socket-timeout-millis: 600000         # 读响应超时；超时边界在 route 阶段
  routes:
    user-provider:
      path: /user-provider/**             # 匹配前缀
      serviceId: user-provider           # 交给服务发现解析成实例列表
      strip-prefix: false                # 保留前缀，否则上游带 context-path 时 404
      sensitiveHeaders: Cookie,Set-Cookie,Authorization  # 外发时剥掉会话凭证
```
注释里的"为什么"：serviceId 与 url 是两条不同的路，只有前者会走 Eureka 的实例列表；strip-prefix 默认 true，若上游服务自己配了 context-path 又剥掉前缀，直接 404。

**拆解**：

### 内外分界：外部扛连接，内部做服务级路由

网关解决的是横切重复：每个服务都要做鉴权、路由、限流、日志，与其在 N 个仓库各写一遍，不如收敛到一个入口，顺带把很多 Provider 的不同端口统一成网关的一个端口、把跨域处理收在一处。但"一个入口"其实是两层。外部网关是 Nginx / OpenResty，它面对的是公网连接数、TLS 握手、keepalive、静态资源与慢客户端，配套配置里 upstream zuul 挂着长连接、proxy_read_timeout 控制等待上游的时间，它擅长用 epoll 与少量 worker 扛住海量连接，却不该知道"user-provider 此刻有几个实例"。内部网关是 Zuul，它自身也是注册到 Eureka 的微服务，靠 @EnableZuulProxy 开启，负责把 /user-provider/** 这样的路径映射到具体服务、并在转发前做业务横切。原书把两者分别叫外部网关与内部网关，分工一句话：一个扛连接与静态流量，一个做服务级路由与业务横切。

为什么不能合并成一层？Nginx 跑在 Lua 协程里，适合连接层与通用限流，但每加一条业务校验都要改 OpenResty 脚本并发版；Zuul 跑在 JVM 里，能用 Java / Groovy 动态读取、编译、运行过滤器，复用 Spring 生态，却不该把宝贵线程耗在慢连接与静态文件上。两层各守住一段，才有"改鉴权只发网关、扩连接只动 Nginx"的独立演进；把它们压成一层，任何一侧的容量问题都会直接传导到另一侧。

代价也必须写清楚：Zuul 成了唯一入口，就是单点（single point of failure）。可用性的答案是它自己不留状态——鉴权结果放请求头、会话放外部 Redis，于是 Zuul 可以横向起多实例；Nginx 用 upstream 做负载均衡与健康检查，挂掉一个实例只影响在途请求；再叠一层 Hystrix 或限流让网关在下游雪崩时先自保，而不是被拖死。判断标准是"任意一台网关进程消失，是否只是一次重试的事"；如果答案是"要人工改配置"，说明状态还粘在网关里，分界就没做干净。

### 四类过滤器、执行顺序与错误归属

Zuul 的大部分功能本身就是过滤器实现的，连"转发"也是 route 类型过滤器。四类标准类型定义在 FilterConstants 里：PRE_TYPE、ROUTE_TYPE、POST_TYPE、ERROR_TYPE，另外还允许自定义类型（例如直接在网关生成响应、不转发上游的 echo 类）。请求先走 pre（路由前，做校验、加头、记调试信息），再走 route（构造并发送请求到上游），拿到结果后走 post（加工响应、收集指标、回写客户端）；任意阶段抛异常，就改走 error 阶段，error 过滤器同样能把最终结果返回给客户端。自定义过滤器继承 ZuulFilter，实现四个契约：filterType() 返回阶段，filterOrder() 返回顺序（值越小优先级越高），shouldFilter() 返回是否生效，run() 写逻辑。把"是否生效"独立成一个方法很关键：黑名单过滤器可以只在 URI 命中某前缀时才跑，避免每条请求都付一遍查库成本；也可以先判断 ctx.sendZuulResponse()，若前面的过滤器已经终止了请求，本过滤器直接返回 false 不再空转。

过滤器之间不直接通信，靠 RequestContext.getCurrentContext() 拿到每请求唯一的上下文。pre 里判定不通过时调用 ctx.setSendZuulResponse(false)，这个标志位会被几乎所有内置过滤器检查——例如 RibbonRoutingFilter 的 shouldFilter() 里就含 ctx.sendZuulResponse() 判断，为 false 就不再发起调用。于是一次拦截不是抛异常中断，而是"后续 route 集体让路"，pre 自己写好响应体即可。这也是 RequestContext 的职责：传状态、传开关、传请求头（addZuulRequestHeader）、传 serviceId 与 routeHost，而不是传对象引用。正因为状态集中在一处，过滤器才能动态加载、彼此零耦合。

| 过滤器类型 | 时机 | 典型职责 | 失败后果 |
|---|---|---|---|
| pre | 路由之前 | 鉴权、黑名单、参数校验、灰度标、追加 USER-ID 头 | 误放行则越权；误拦截则整条链被 setSendZuulResponse(false) 掐断 |
| route | 转发之时 | serviceId 解析、选实例、发起调用、超时控制 | 超时或选错实例直接抛异常，转入 error 阶段 |
| post | 上游返回之后 | 加响应头、收集指标、改写响应、回写客户端 | 此处再抛异常会进 error，且响应可能已开始写出，处理要幂等 |
| error | 任一阶段异常后 | 统一异常映射、兜底响应、告警 | 自身再异常则无兜底，客户端拿到容器级错误 |

这张表真正的用途是排障：看到"鉴权通过却 404"先查 route 与 strip-prefix，看到"响应头没加上"先查 post 是否被 error 顶掉，看到"重复回写"先查 post 与 error 有没有都写了响应。把四类按"时机 + 失败后果"记，比按方法名记更能定位问题落在哪一段。

### 路由与服务发现、超时重试边界，以及"网关写业务逻辑"的反模式

路由规则有两条路。url 直连把 /blog/** 之类前缀映射到固定上游地址，简单、但不感知实例上下线；serviceId 方式把路径映射到服务名，由 Eureka Client 拉取实例列表、Ribbon 选一个实例转发，这才是动态路由，也是"服务已经下线、消费端还在往它发请求"的解药——实例列表由注册中心维护，网关不必硬编码 IP。配置里 serviceId 与 url 互斥，前者必须配合 @EnableDiscoveryClient（@EnableZuulProxy 已默认导入）。前缀处理是另一个高频坑：strip-prefix 默认 true，会剥掉 /demo-provider 这类前缀；若上游服务自己配了 context-path，剥掉就 404，必须显式设 false。sensitiveHeaders 则相反——它列的是"要被剥掉"的头，默认 Cookie、Set-Cookie、Authorization 不外泄；内部转发想保留原始头，要把该配置置空，而发往外部站点的规则要单独把凭据类头挡住，局部配置覆盖全局配置。这个细节决定了"配了 header 透传却不生效"的原因往往不在代码，而在网关替你把头清掉了。

超时与重试的边界在 route 阶段：connect-timeout-millis 管建连、socket-timeout-millis 管等响应，两者都只约束"网关等下游"这一段，不约束 Nginx 等网关、也不约束客户端等整条链——把它们调大只是让网关多等一会儿，不会让下游更快，反而会占住更多线程。重试必须建立在一个前提上：被重试的操作幂等，否则一次超时重放会变成两次下单；而且重试只能在 route 内做，post 已经拿到响应后再重试等于重复回写。把 ribbonIsolationStrategy 设为 THREAD，是让命令在独立线程池里执行，超时才不会吃掉容器线程；这与信号量隔离的取舍在于是否值得为硬超时付出线程切换与上下文传递的代价。

为什么"网关里写业务逻辑"是反模式？因为网关是所有服务共用的基础设施，一旦把订单状态判断、库存扣减这类规则塞进 run()，它就必须跟着业务一起发版，任何一条业务线改动都要全量重启入口；不同服务对同一路径的诉求会挤进同一个过滤器，退回"改一处动全身"。更隐蔽的代价是可移植性：Provider 被迫依赖网关约定的头与上下文，换个网关（甚至换成 OpenResty）就跑不起来。判断标准很直接——凡是"换个业务域就要改"的代码，都不该出现在网关；网关只放"换个业务域也不变"的横切规则。

### 今天怎么做：Spring Cloud Gateway 与 Zuul 1 的 I/O 模型差别

今天怎么做：Zuul 1 已进入维护模式，现代替代是 Spring Cloud Gateway，构建在 WebFlux 之上，用 GlobalFilter 与链式调用（chain）取代 ZuulFilter 的四阶段。变的先是 I/O 模型：Zuul 1 是 Servlet 阻塞模型，每个在途请求占住一个容器线程，高并发下线程数与上下文切换就是天花板；Spring Cloud Gateway 基于事件循环与非阻塞 I/O，少量线程即可维持大量连接，更适合长连接与高扇出。其次是表达形式：四类型 + filterOrder 被统一的 GlobalFilter + 顺序接口取代，pre/route/post 的边界被"链前处理 / 转发 / 链后处理"重新表达，路由同样由服务发现解析成实例。不变的是职责边界：单一入口、路由到服务、横切前置、错误统一兜底、网关自身无状态可横向扩展——这些一条都没变，所以理解了 Zuul 的过滤器链，读 Gateway 的过滤链只是换一套名词。

选型判断也不该只看性能数字：如果存量过滤器全是 ZuulFilter、且流量远未到线程瓶颈，迁移成本可能高于收益；如果目标是长连接、高并发或要与响应式下游打通，WebFlux 模型才真正解决问题。把"换组件"当成"换职责"，是这次映射里最该避免的误读。原书当年推荐 Zuul 用于一般生产场景、高并发用 Gateway，理由也正是 I/O 模型的差别，而不是功能多寡。

<div class="chapend" markdown="1">

<div class="chapend" markdown="1">

**容易误解的地方**：

- 很多人以为网关只是"转发一下"，其实 Zuul 自身的大部分功能都由过滤器实现、连路由也是 route 过滤器，因为转发只是这条链上的一环。
- 很多人以为 filterOrder 越大越先执行，其实值越小优先级越高，因为同类型内按该值升序排列。
- 很多人以为 error 过滤器只是"记个日志"，其实它能把最终结果返回客户端，因为异常会改走 error 阶段而非原阶段收尾。
- 很多人以为 sensitiveHeaders 配了就会把头传给下游，其实它列的是"要被剥掉"的头，因为外发时不能泄露 Cookie / Authorization 这类会话凭证。

**记住这一句**：网关的价值是把横切逻辑收成一条链，风险是把全站流量压在一个必须永远在线、又必须无状态的进程上。
</div>
</div>
