# 第9章 身份识别：Spring Security、JWT 与会话共享

**一句话**：认证归网关、鉴权归服务；无状态令牌换来扩展性，代价是主动失效能力。

**为什么需要它**：登录态原先放在单体的 HttpSession 内存里，拆成 8 个 Provider 之后，用户在 A 实例登录、下一个请求落到 B 实例就被判未登录；给 Nginx 开了粘滞会话（sticky session），结果一台实例重启，上面几千个会话一起丢失。改成 Redis 集中存 Session 后确实不丢了，但每个 Provider 每次请求都要回 Redis 取一次用户信息，网关和 Provider 还都被迫依赖同一份会话结构——只要 UAA 往 Session 里加一个字段，全线都要跟着改。更要命的是一次越权：网关验完 token 就把原始令牌透传下去，某个 Provider 直接信任了这个头，而该头在内网其实可以被伪造。这一章把三件事分开：认证（authentication，"你是谁"）与鉴权（authorization，"你能做什么"）的分界、Spring Security 在原理层怎么把凭证放进上下文、JWT 与分布式 Session 各自把状态放在哪、以及网关验签之后身份该以什么形式、在怎样的信任边界（trust boundary）上传给下游。

**核心结构**：

| 概念 | 作用 |
|---|---|
| 认证 authentication / 鉴权 authorization | "你是谁" / "你能做什么"，是过滤器链上两个不同环节 |
| Spring Security 过滤器链 | 请求进业务前逐层拦截、从请求头组装凭证、决定放行 |
| SecurityContext / SecurityContextHolder | 线程级容器，持有本次请求的认证结果 |
| Authentication | 凭证/令牌：principal、credentials、authorities、isAuthenticated |
| AuthenticationManager / ProviderManager | 认证入口；后者维护 provider 列表，找"支持该令牌"的那个 |
| AuthenticationProvider | supports() 判定能否处理、authenticate() 执行校验 |
| UserDetails / UserDetailsService / GrantedAuthority | 数据源里的用户（密文密码、是否可用）、加载接口、权限集合 |
| JWT：header.payload.signature | 三段 Base64 + 签名，声明（claims）放在 payload |
| 分布式 Session / Spring Session / Redis | 把会话从 Web 容器剥离，交给外部存储共享 |
| UAA 用户账号与认证服务 | 登录时校验密码、签发令牌 |
| USER-ID 请求头 | 网关验签后只把用户标识透传给下游 |

**关键设计**：

```text
  [登录]
  浏览器 --用户名/密码--> UAA(账号与认证) --校验--> 用户库
                              |
                              +--> 分布式 Session(Redis)：用户ID/角色/盐
                              +--> 签发 JWT：header.payload.signature
  浏览器 <-- JWT(存 localStorage，放 Authorization / token 头) --+

  [每次请求]
  浏览器 --带 JWT--> Nginx --> Zuul(pre 过滤器 + Spring Security)
                                  | 取头 -> 组装 Authentication
                                  | AuthenticationManager 找 Provider
                                  | subject 取 Session ID -> Redis 取盐 -> 验签
                                  v 通过：SecurityContext.setAuthentication(...)
                              pre 过滤器 addZuulRequestHeader("USER-ID", ...)
                                  |  （不透传原始 token）
                                  v
                           Provider：按 USER-ID 鉴权 / 做业务，不重复认证

  信任边界：网关之外不可信；网关到 Provider 是否可信，决定要不要再验一次
```

```text
  JWT 三段（示意）
  header  {"typ":"JWT","alg":"HS256"}        -> Base64
  payload {"sub":"...","exp":...,"iat":...}  -> Base64（明文，不放敏感信息）
  signature = HMAC256( base64(header) + "." + base64(payload), salt )
  验签：用同一 salt 重算签名比对 -> 一致才说明未被篡改
```
注释里的"为什么"：payload 只编码不加密，签名保证的是"没被改过"，不是"读不到"；所以密码、密钥绝不能放进去。

**拆解**：

### 认证与鉴权的分界，以及 Spring Security 的过滤器链

先把两个词钉死：认证（authentication）回答"你是谁"，产出的是一个已验证的身份；鉴权（authorization）回答"你能做什么"，拿这个身份去比对权限集合。二者在 Spring Security 里是不同环节——把凭证放对地方只是第一步，anyRequest().authenticated() 之外的权限判断才是第二步。对应到微服务：网关负责认证（token 是不是有效、是不是过期、是不是本人），下游服务负责鉴权（这个用户能不能改这条订单）。把鉴权也全压在网关上，网关就得懂每个服务的业务规则，又退回到"网关写业务逻辑"的反模式；反过来让每个 Provider 各自验一遍 token，则是重复验证，既降性能又拖累可移植性。

原理层只需要记住一条链和三个对象。请求进 Web 容器后先过一串过滤器，某个 OncePerRequestFilter 从 HTTP 头里取出 token、组装成 Authentication 令牌，交给 AuthenticationManager；它的实现 ProviderManager 维护一个 AuthenticationProvider 列表，用 supports() 找出能处理该令牌的提供者，再调 authenticate() 校验，成功后把结果写进 SecurityContextHolder 持有的 SecurityContext，失败则清上下文并交给失败处理器返回 403。认证通过后业务代码在任何地方都能从上下文取到当前身份，靠的就是这个线程级容器；也正因为它是线程级的，跨线程异步编排时身份不会自动带过去，这是响应式链路里必须显式传递上下文的根因。

三个对象的职责要分清：Authentication 是"待验证或已验证的凭证"，含 principal（身份，可以是用户名或 UserDetails）、credentials（密码）、authorities（GrantedAuthority 权限集合）、isAuthenticated；UserDetails 是数据源里的用户，含密文密码、账号是否过期/锁定/可用；UserDetailsService.loadUserByUsername() 负责按名加载，PasswordEncoder 负责明文与密文匹配——原书特别点明这是"编码（Hash）"而非"加密"，因为不可逆，数据库泄露也看不到明文，忘记密码只能重置。DaoAuthenticationProvider 把"请求里的明文"和"数据源里的密文"用同一个 PasswordEncoder 对上，整个认证的信任根就落在这里；两处若用了不同编码器，匹配必然失败，这也是"配了却认证不过"的常见原因。原书还给出一个工程判断：建议只在网关启用 Spring Security，集群内部 Provider 不启用，要关闭自动配置，排除 SecurityAutoConfiguration 即可。

### Session 在单体成立、在微服务成为负担

HTTP 本身无状态，服务端要认出"还是这个用户"，就得存一份会话并给客户端一个 Session ID。单体里这份状态放内存最自然，零网络开销、失效就是删掉对象。一旦横向扩展成多实例，麻烦来自三个方向：粘滞会话（sticky session）把用户绑到固定实例，实例一重启会话全丢，扩容还会导致分布不均；会话复制让每个实例都持有一份，节点越多同步写放大越严重，且复制窗口内各实例视图可能不一致；集中式 Session 把状态挪到 Redis（Spring Session 用 springSessionRepositoryFilter 把 HttpSession 换成 RedisSession，由 SessionRepository / RedisOperationsSessionRepository 读写），确实解决"丢"，但每次请求多一次网络 I/O，且网关与所有 Provider 被迫共享同一份会话结构，耦合度随节点数上升。

Spring Session 的设计值得单独记：它抽象出自己的 Session 接口、再用适配器适配 Servlet 的 HttpSession，是为了同时覆盖 HTTP、WebSocket、非 Web 等传输与多种存储；RedisSession 里 cached 做本地缓存优先读、delta 只跟踪变化的属性、saveDelta() 只持久化差异，都是在压那一次网络 I/O 的成本。但这些都只是"把负担挪走"，没有消除状态本身；只要状态还在服务端，"谁能改它、改了谁要跟着变"就始终是架构问题。

| 方案 | 状态存哪 | 失效能力 | 适用面 |
|---|---|---|---|
| 单体内存 Session | 单进程内存 | 强：删对象即失效 | 单实例、无横向扩展 |
| 粘滞会话 | 绑定到某实例内存 | 中：实例宕机即丢 | 小规模、能容忍重登 |
| 会话复制 | 每个实例各一份 | 中：同步存在窗口 | 节点少、写不频繁 |
| 集中式 Redis Session | 外部 Redis | 强：删 key 即全局失效 | 节点多、要即时踢人 |
| 自包含 JWT | 不存服务端（在令牌里） | 弱：到期前无法主动作废 | 高扩展、可容忍短时无法注销 |

这张表的判断顺序是：先问"要不要即时踢人、改权限立刻生效"，要，就别选纯无状态；再问"节点规模与 I/O 预算"，大就别选复制；最后才在集中式与令牌之间权衡。原书那套 UAA 方案其实是混合态：JWT 的 subject 存的是 Session ID，网关验签时仍要回 Redis 取盐、查会话是否存在，甚至比对令牌字符串来拦重复登录——它用无状态外壳换扩展，却把失效能力留在了服务端。这也说明"无状态"不是非黑即白，而是把状态放在链路哪一段的取舍。

### JWT 的三段、验签，以及无状态的收益与代价

JWT（JSON Web Token，RFC 7519）是一种凭证编码规范，用点号把三段连起来：header（typ、alg）声明类型与算法，payload 放声明（claims），signature 是把前两段的 Base64 用 header 指定算法加盐（salt）算出的签名。原书用 Algorithm.HMAC256 签发、JWT.require(...).build() 构造验证器再 verify(...) 演示验签；校验方用同一 salt 重算签名比对，一致才说明内容未被篡改。标准声明有 iss（签发人）、sub（主题）、aud（用户）、iat（签发时间）、exp（过期时间）、nbf（此前不可用），还能加 uid、user_name 这类私有声明。

收益来自"自包含"：网关验签不必回查用户库，Provider 不必共享会话，水平扩展时加实例不用同步任何状态，天然适合单点登录（SSO）与多网关。代价同样来自自包含，且每一条都要提前认账。第一，无法主动失效：令牌在 exp 之前一直有效，注销、改密、踢人都不能立刻生效，要补黑名单或退回集中式 Session——这正是上一节对比表里"弱"的来源。第二，令牌体积：私有声明越塞越多，每个请求都要多带一段 Base64，且 payload 是编码后的明文，只防篡改不防读取，绝不能放密码、密钥、敏感信息。第三，密钥轮换（key rotation）：salt 一换，旧签名的令牌全部验不过，必须设计并行验证旧密钥的过渡期，否则等于全量强制下线；原书把用户密码密文当盐，好处是改密即自然作废旧令牌，坏处是密钥与用户数据绑死、轮换粒度很粗。第四，过期时间与刷新令牌（refresh token）：exp 设长了风险高、设短了用户反复登录，所以配套 oauth 客户端配置里才有 access_token_validity 与 refresh_token_validity 两个时长，用短时访问令牌换安全、用长时刷新令牌换体验，刷新令牌本身也要能被服务端作废。

把这些代价摆在一起，判断标准就清楚了：JWT 不是"比 Session 先进"，而是拿"主动失效能力"换"免状态同步"。业务要求即时注销、权限实时生效时，纯无状态是错误选择；能容忍一个短过期窗口、又需要多网关水平扩展时，它才划算。

### 信任边界与今天怎么做：身份怎么传给下游、内部调用要不要带身份

网关验签通过之后，身份怎么到 Provider 手里，是这一章最容易被忽略的设计点。原书列了三种：透传整个 JWT、只透传 subject 里的 Session ID、或把会话里缓存的用户 ID 取出来加一个 "USER-ID" 头。前两种都要求 Provider 与网关共享同一套会话/密钥体系，而负责认证的网关未必总是 Zuul——换成性能更高的 OpenResty 甚至 Kong 时，共享 Session 的难度陡增；原书因此推荐第三种，并用一个 pre 类型的过滤器在 run() 里 addZuulRequestHeader 完成。这个选择的本质不是"哪种头更短"，而是把下游从"必须懂令牌格式"里解耦出来：Provider 只认一个用户标识，不认签发算法、不认盐、不认会话结构。

信任边界就在这里：网关之外是不可信的公网，必须验签；网关到 Provider 这一段是否可信，决定"下游还要不要再验一次"。请求头透传的本质是"下游信任网关已经验过"，一旦这个头在内网可被绕过直连伪造，信任就崩了——所以要么在网络层保证 Provider 只能经网关到达，要么让下游自行验签（代价是每个服务都绑定密钥、回到重复校验）。两种不是对错，是信任边界画在哪、以及愿不愿意为它付网络隔离的成本。把 USER-ID 这种"可被任意构造的明文标识"当成鉴权依据，前提永远是"这个头只可能来自可信的网关"。

随之而来的是服务提供者之间的会话共享关系。原书对比了两种：全局共享（所有网关与 Provider 共用一个分布式 Session，节点少时可行，规模一大 I/O 成瓶颈且耦合极高）、局部按需共享（网关与 Provider 不再传 Session ID，改传 User ID，需要会话的服务再按 User ID 去 Redis 反查绑定关系）。后者给了那个关键判断：内部调用是否还需要带用户身份，取决于业务——只有当下游要按用户做鉴权、审计或数据隔离时才需要透传；纯内部的服务间调用、定时任务、异步消息消费不带用户身份反而更干净，因为带身份等于重新把用户上下文耦合进每一跳。判断标准一句话：谁要按"人"做决策，谁才需要身份。

今天怎么做：把认证从各服务里彻底收走，交给 OAuth2 / OIDC 的授权服务器统一签发，网关只做令牌校验与路由，下游作为资源服务器按 scope 与权限鉴权；对外用短时访问令牌 + 刷新令牌缓解"无法主动失效"，对内走向零信任（zero trust）——服务间调用也带可验证的服务身份，而不是默认信任内网；要即时踢人，就把失效判断收敛到一个轻量黑名单或足够短的 exp 上。变的是令牌格式与校验位置，不变的是那条边界：认证在入口一次做完，身份以最小必要形式向下游传递，鉴权留在离业务最近的地方。

<div class="chapend" markdown="1">

<div class="chapend" markdown="1">

**容易误解的地方**：

- 很多人以为认证通过就等于有权限，其实认证只回答"你是谁"、鉴权另由 authorities 决定"你能做什么"，因为二者是过滤器链上不同环节。
- 很多人以为 JWT 的 payload 是加密的，其实它只是 Base64 编码、等同明文，签名只防篡改不防读取，所以不能放密码等敏感信息。
- 很多人以为用了 JWT 就能随时让令牌失效，其实自包含令牌在到期前服务端无法主动作废，要失效就得补黑名单或退回集中式 Session，因为验签不查库。
- 很多人以为网关把原始 token 原样透传最省事，其实更稳的是只透传一个下游能自证的用户标识，因为透传 token 等于让所有 Provider 绑定同一套密钥与会话体系。

**记住这一句**：认证解决"是谁"，鉴权解决"能做什么"；把状态拿掉换来的扩展能力，要靠短时令牌、信任边界和最小透传补回来。
</div>
</div>
