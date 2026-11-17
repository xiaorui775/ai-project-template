# 后端编码规范（Spring Boot 3 + Spring Cloud Alibaba）

> 能被 Checkstyle / Spotless / 编译器强制的内容交给配置，本文件写**工具管不到但必须遵守**的部分。

## 1. 模块划分（Maven 多模块）

```
pom.xml                      # packaging=pom，dependencyManagement + modules
├── services/common          # 与业务无关的基础设施（Result、异常、脱敏、工具、starter）
├── services/gateway         # 网关（不依赖 common 的业务部分，只引 web 基础）
└── services/<domain>/       # 业务服务
    ├── <domain>-api         # 对外契约：DTO/VO/Feign 接口（被其他服务依赖，只含接口与模型）
    └── <domain>-server      # 实现：Controller/Service/Mapper（依赖自己的 -api）
```

**规则**：

- 依赖方向单向：`server → api`，`服务A-server` 只能依赖 `服务B-api`，**禁止依赖 `服务B-server`**。
- `common` 中禁止出现业务实体（Order、User 等）。
- 子模块 `<dependency>` 中**不写 `<version>`**（统一由根 POM 管理），新增依赖先加到根 POM。
- 新增第三方依赖必须查 license 与 CVE，并在 PR 说明。

## 2. 包结构

```
com.company.<domain>
├── api/            # 只放 Feign 接口 + 模型（-api 模块）
├── controller/
├── service/        # 接口
├── service/impl/
├── mapper/         # MyBatis-Plus Mapper
├── entity/         # 与表一一对应
├── dto/            # 入参
├── vo/             # 出参
├── convert/        # MapStruct
├── config/
├── client/         # 对其他服务的 Feign 客户端 + fallback
├── constant/
└── job/
```

## 3. 标准模板

### Controller

```java
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
@Tag(name = "订单")
public class OrderController {

    private final OrderService orderService;

    @GetMapping("/{id}")
    @Operation(summary = "查询订单详情")
    public Result<OrderVO> detail(@PathVariable Long id) {
        return Result.ok(orderService.getDetail(id));
    }

    @PostMapping
    @Operation(summary = "创建订单")
    public Result<Long> create(@RequestBody @Valid OrderCreateDTO dto) {
        // Controller 不写业务，只透传；当前用户由 UserContext 提供
        return Result.ok(orderService.create(dto));
    }
}
```

### Service

```java
@Override
@Transactional(rollbackFor = Exception.class)
public Long create(OrderCreateDTO dto) {
    // 1. 幂等：业务唯一键
    Order exist = orderMapper.selectByBizNo(dto.getBizNo());
    if (exist != null) {
        return exist.getId();     // 幂等返回，不抛错
    }
    // 2. 批量取商品信息（一次远程调用，禁止在循环里调 Feign）
    List<Long> skuIds = dto.getItems().stream().map(OrderItemDTO::getSkuId).toList();
    Map<Long, SkuVO> skuMap = skuClient.batchGet(skuIds).getData().stream()
            .collect(Collectors.toMap(SkuVO::getId, Function.identity()));
    for (OrderItemDTO item : dto.getItems()) {
        SkuVO sku = skuMap.get(item.getSkuId());
        if (sku == null) {
            throw new BizException(ErrorCode.SKU_NOT_FOUND);
        }
        // 价格用服务端快照，禁止信任前端传入的金额
        item.setPrice(sku.getPrice());
    }

    // 3. 落库
    Order order = OrderConvert.INSTANCE.toEntity(dto);
    order.setStatus(OrderStatus.CREATED);
    orderMapper.insert(order);
    log.info("创建订单成功 orderId={} userId={}", order.getId(), UserContext.userId());
    return order.getId();
}
```

### 跨服务调用（Feign）

```java
@FeignClient(
    name = "sku-service",
    path = "/internal/skus",
    fallbackFactory = SkuClientFallbackFactory.class,
    configuration = FeignTimeoutConfig.class)
public interface SkuClient {

    @PostMapping("/batch")
    Result<List<SkuVO>> batchGet(@RequestBody List<Long> skuIds);
}

@Component
@Slf4j
public class SkuClientFallbackFactory implements FallbackFactory<SkuClient> {
    @Override
    public SkuClient create(Throwable cause) {
        return skuIds -> {
            log.warn("调用 sku-service 失败，降级：无库存信息，skuIds={}", skuIds, cause);
            // 关键：降级不能返回“空=成功”，要显式失败或标记为不可判定
            throw new BizException(ErrorCode.DEPENDENCY_UNAVAILABLE, "商品服务暂不可用");
        };
    }
}
```

### 跨服务写事务（Seata，谨慎使用）

```java
/**
 * 跨服务写事务：本地写订单 + 远程扣减库存。
 * 回滚点：库存扣减后任一步失败 → Seata AT 回滚 inventory 的 undo_log。
 * 幂等键：bizNo（唯一索引 uk_biz_no）。
 * 注意：库存服务必须建 undo_log 表，且库存扣减需用 UPDATE ... WHERE stock >= n 防超卖。
 */
@GlobalTransactional(name = "order-create", rollbackFor = Exception.class, timeoutMills = 30_000)
@Transactional(rollbackFor = Exception.class)
public Long create(OrderCreateDTO dto) { … }
```

## 4. 必须遵守的细则

| 项 | 规则 |
|---|---|
| 返回值 | 统一 `Result<T>`；`Result.ok(data)` / `Result.fail(ErrorCode)` |
| 异常 | `BizException(ErrorCode)`；禁止裸 `RuntimeException` |
| 事务 | 写操作 `@Transactional(rollbackFor = Exception.class)`；禁止在 `private` 方法上加事务；同类内部调用不走代理 |
| 事务与远程调用 | `@Transactional` 内做远程调用时，尽量把远程调用挪到事务外，或用 Seata |
| SQL | 只用 `#{}`；显式列名；禁止 `SELECT *`；批量用 `saveBatch`（每批 500） |
| 时间 | 统一 `LocalDateTime`，DB 存 `DATETIME(3)`；禁止 `java.util.Date` |
| 金额 | `BigDecimal`，`scale=2`，比较用 `compareTo`，禁止 `equals` |
| ID | 雪花 ID（`@TableId(type = IdType.ASSIGN_ID)`），前端接收用 `string` 防精度丢失 |
| 分页 | 单页 ≤ 500；导出接口单次 ≤ 5000 且异步 |
| 缓存 | Key 前缀 `{service}:{entity}:{id}`；写库后删缓存；缓存值必须有 TTL |
| 配置 | `@ConfigurationProperties` 优先；`@Value` 只用于单值；敏感项走环境变量 |
| 日志 | `@Slf4j`；入口 `info` 带 traceId/userId；禁止打印完整对象 |
| 注释 | 公共类/接口方法写 Javadoc（中文）；**复杂业务分支必须写"为什么"**，不写"做了什么" |

## 5. 常见错误（AI 最容易犯）

1. Controller 里写 `try/catch` 然后 `return Result.fail(e.getMessage())` —— 丢掉异常类型与堆栈。
2. 在 `@Transactional` 方法里调 Feign，且没有 Seata，导致远程成功本地回滚 → 数据不一致。
3. Feign fallback 返回空 List，被上层的 `.get(0)` 触发 NPE，或静默"成功"。
4. MyBatis 用 `${}` 拼排序字段 → SQL 注入。
5. `@Transactional` 加在 private/同类自调用方法上，事务不生效。
6. 循环里查库（N+1），接口在压测下直接打满连接池。
7. 用 `LocalDateTime.now()` 直接写库，多实例时钟漂移 → 统一由 DB `CURRENT_TIMESTAMP(3)` 或注入 `Clock` Bean。
8. 在 `common` 里加了业务类的 import，导致模块反向依赖。

## 6. 新增一个服务的检查清单

- [ ] 模块命名 `<domain>-api` / `<domain>-server`，已在根 POM `<modules>` 注册
- [ ] `application.yml` 只含默认值，环境相关走环境变量，新增变量同步 `.env.example`
- [ ] `bootstrap.yml` 指向 Nacos，命名空间按环境区分
- [ ] 引入 `common-spring-boot-starter`（统一异常/Result/traceId）
- [ ] 建 `undo_log` 表（若参与 Seata 事务）
- [ ] 写 `db/migration/V1__init.sql`（Flyway）
- [ ] 网关路由（Nacos 配置）、Sentinel 规则、接口文档分组
- [ ] README 段落 + `docs/architecture.md` 服务表更新