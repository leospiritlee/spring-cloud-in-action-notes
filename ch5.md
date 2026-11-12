# 第5章 Feign 动态代理：一个接口怎么变成一次 HTTP 调用

**一句话**：一个只写了注解的接口，被 JDK 动态代理包成 `InvocationHandler`，方法调用时现拼出 HTTP 请求并交由 `Client` 发出——三套超时（Feign/Ribbon/Hystrix）里谁小谁先说话。

**为什么需要它**：你只写了 `@FeignClient("order-service") interface OrderApi { @GetMapping("/{id}") Order get(@PathVariable Long id); }`，没有实现类，却能在注入后直接 `orderApi.get(1L)` 拿到远程结果。魔法全在"动态代理"。现场：你给 Feign 配了 3 秒 `readTimeout`，压测还是 1 秒就熔断——因为真正兜底超时的是 Hystrix。现场二：你想给所有请求统一加鉴权头，不知道该在哪一环插手——答案是 `RequestInterceptor`。现场三：你怀疑请求 URL 拼错，却找不到"拼接逻辑在哪"——它散落在 `Contract`、`RequestTemplate`、`Encoder` 之间。这一章把这条流水线从"接口"到"字节"逐段拆开。

**核心结构**：

| 组件 | 职责 |
|---|---|
| `Contract` | 解析接口上的注解，生成 `MethodMetadata`（URL 模板、HTTP 方法、参数索引） |
| `RequestTemplate` | 一次调用的可变的请求模板（URL、Query、Header、Body） |
| `Encoder` / `Decoder` | 把对象编成请求体、把响应体解回对象 |
| `RequestInterceptor` | 在发出前统一加工请求（加 Header、加 Token） |
| `InvocationHandler`（ReflectiveFeign） | JDK 动态代理的调用入口，把方法调用路由到 `MethodHandler` |
| `MethodHandler`（SynchronousMethodHandler） | 每个方法一个，负责拼模板、拦截、调用 `Client` |
| `Client` | 真正发 HTTP 的实现（默认 `HttpURLConnection`，可换 OkHttp/Apache） |
| `Target` | 把 serviceId 解析成基础 URL（结合负载均衡） |

**关键设计**：

```java
// 示意：Feign 代理的骨架（非原书逐字，仅为说明层级）
public class ReflectiveFeign implements InvocationHandlerFactory {
  // 每个接口方法 → 一个 SynchronousMethodHandler
  public Object invoke(Object proxy, Method method, Object[] args) {
    MethodHandler handler = dispatch.get(method);   // 按方法路由
    return handler.invoke(args);                     // 交给对应处理器
  }
}

// SynchronousMethodHandler.invoke 的核心步骤
RequestTemplate template = targetRequest(args);      // 1. 用 Contract 生成的元数据拼模板
template = applyInterceptors(template);              // 2. RequestInterceptor 统一加头/改写
Client client = loadBalancedClient();                // 3. 带 Ribbon 的 Client：先选实例
Response resp = client.execute(template.request());  // 4. 真正发 HTTP（serviceId→具体地址）
return decode(resp);                                 // 5. Decoder 把响应解回对象 / 走 errorDecoder
```

注意第 3 步：当 Feign 与 Ribbon 配合时，`Client` 是"带负载均衡的"——它内部调用 Ribbon 选出实例，把 `http://order-service/{id}` 换成 `http://10.0.1.7:8080/{id}` 再发。

**拆解**：

### 5.1 为什么必须是接口 + 动态代理
JDK 动态代理要求目标必须是接口，它在运行时生成一个"实现该接口"的代理类，把所有方法调用收口到一个 `InvocationHandler`。Feign 借此做到"你只声明、我不写实现"：编译期不存在任何实现类，运行时才由代理把方法调用翻译成 HTTP。这也意味着——你不能在接口方法里写逻辑，所有行为都来自代理链的组装（Contract 解析、Interceptor 加工、Client 发送）。好处是零侵入、可组合；代价是调试时要理解这条"看不见的链"。

### 5.2 代理实例的创建流程
`@EnableFeignClients` 扫描所有 `@FeignClient` 接口，每个接口由一个 `FeignClientFactoryBean` 负责。它拿到配置（日志级别、拦截器、编解码器、超时 Options 等），构建 `Feign.Builder`，最终 `target()` 生成代理对象放进容器。`Builder` 是可插拔的：换 `Client`（OkHttp）、加 `Logger`、`RequestInterceptor`、自定义 `Contract` 都在这一步完成。理解"创建"和"执行"是两个阶段，就能解释为什么有些配置（如超时）要在创建时就定好、运行时改不动。

### 5.3 一次调用的执行流程
方法被调用 → `InvocationHandler` 按方法找到 `MethodHandler` → `SynchronousMethodHandler` 用 `MethodMetadata` 把参数填进 `RequestTemplate`（Query 参数、Path 变量、Body）→ 依次过所有 `RequestInterceptor`（统一加鉴权头、traceId 等）→ `Client.execute` 发出 → `Decoder` 把 2xx 响应解回对象；非 2xx 走 `ErrorDecoder`（默认对 4xx/5xx 抛 `FeignException`，5xx 才可能触发重试/熔断）。每一步都是独立可替换的组件，这正是 Feign "声明式但高度可定制"的来源。

### 5.4 超时三件套：谁小谁说了算
这是"配了不生效"最高频的坑。叠加 Feign + Ribbon + Hystrix 时存在三套超时：
- **Feign 的 `Options`（connect/read）**：仅当 Ribbon 未介入、使用默认 `Client` 时直接生效；一旦走 Ribbon，`Client` 实际用的是 Ribbon 的 `ConnectTimeout`/`ReadTimeout`。
- **Ribbon 的 `ConnectTimeout`/`ReadTimeout`**（默认 1000/1000ms 量级）：真正控制底层 socket 超时的，常常是它。
- **Hystrix 的 `execution.timeout`**（默认 1000ms）：它在最外层包住整个调用，超时直接走 fallback。

结论：**Hystrix 超时 < Ribbon 超时 < Feign 超时** 时才不浪费；若 Hystrix 超时最小，调用常常没等到底层超时就被熔断了——你以为"3 秒超时"，其实 1 秒就 fallback。同理 Ribbon 的重试次数 × 单次超时 也得小于 Hystrix 超时，否则重试还没跑完就被熔断截断。

**今天怎么做**：OpenFeign + Spring Cloud LoadBalancer 仍是主流组合，超时层级（Feign Options / LoadBalancer 配置 / Resilience4j `TimeLimiter`）依然存在且必须按大小关系排布；`RequestInterceptor` 仍是统一加认证/链路追踪头的标准位置。响应式场景下 Feign 被 WebClient/声明式 HTTP 客户端替代，但"声明 → 拼请求 → 拦截 → 发送 → 解码"的流水线结构没变。

<div class="chapend" markdown="1">

<div class="chapend" markdown="1">

**容易误解的地方**：

1. 以为"给 Feign 配了 readTimeout 就一定生效"——走 Ribbon 时实际是 Ribbon 的 socket 超时在起作用。
2. 以为"超时配大一点更安全"——在 Hystrix 包裹下，Hystrix 超时最小，配大 Feign 超时只是让线程被占用更久，不解决熔断。
3. 以为"动态代理是 Feign 自己实现的黑魔法"——它就是标准 JDK `Proxy` + `InvocationHandler`，接口是唯一前提。
4. 以为"加鉴权头要改每个方法"——用 `RequestInterceptor` 在调用前统一加，才是正解。

**记住这一句**：Feign 把"方法调用"翻译成"HTTP 请求"靠的是 JDK 动态代理 + 一条可替换组件的流水线；三套超时里最小的那一个，才是你真正体验到的超时。
</div>
</div>
