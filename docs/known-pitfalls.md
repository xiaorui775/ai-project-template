# 踩坑清单（Known Pitfalls）

> 作用：把"已经踩过的坑"固化下来，AI 和人都会反复踩。新增一条 = 省掉一次线上事故。
> 格式：现象 → 根因 → 正确做法 → 相关代码位置。

## 前端

### P-01 axios 响应拦截器里 return `res` 而不是 `res.data`
- **现象**：调用方拿到 `{data: {...}}` 双层，类型对不上但编译通过。
- **根因**：拦截器返回了完整 AxiosResponse。
- **正确**：统一在 `packages/api-client/src/http.ts` 拆包，业务侧只拿 `Result<T>`。

### P-02 Element Plus 表单 `model` 直接绑定 props 对象
- **现象**：父组件状态被意外修改。
- **正确**：`const form = reactive({ ...props.value })` 深拷贝，提交后再 emit。

### P-03 大 ID 前端精度丢失
- **现象**：`orderId: 1893456789012345678` 传到前端变成 `...678000`。
- **根因**：JS Number 只有 53 位精度。
- **正确**：所有雪花 ID 后端序列化为 **string**（`@JsonSerialize(using = ToStringSerializer.class)`），前端类型用 `string`。

### P-04 路由切换后旧请求回写状态
- **现象**：快速切菜单，页面显示的是上一个页面的数据。
- **正确**：请求带 `AbortController`，`onUnmounted` 时 abort；或请求序号比对。

## 后端

### P-05 `@Transactional` 加在 private / 同类自调用方法上，事务不生效
- **根因**：Spring AOP 代理机制。
- **正确**：放在 public 且由外部代理调用；同类调用用 `AopContext` 或拆到另一个 Bean（优先拆 Bean）。

### P-06 `@Transactional` 方法内调用 Feign，远程成功本地回滚 → 数据不一致
- **现象**：库存扣了，订单没落库。
- **正确**：要么用 Seata `@GlobalTransactional`，要么把远程调用移出本地事务（本地先写"待确认"，成功后回调确认）。**优先后者**。

### P-07 Feign fallback 返回空集合
- **现象**：商品服务挂了，订单页显示"无商品"，用户以为数据没了。
- **正确**：降级要"显式失败"或"显示缓存 + 提示"，禁止静默返回空。

### P-08 Nacos 配置里写了 `spring.datasource.password`
- **现象**：配置中心能被人看到明文密码。
- **正确**：敏感项走环境变量 / KMS；Nacos 只放非敏感可变项。

### P-09 MyBatis `${}` 拼接排序字段
- **现象**：`order by ${sortField}` 被注入 `1; drop table ...`。
- **正确**：白名单 map 映射，非法值直接拒绝。

### P-10 Seata 参与者没建 `undo_log` 表
- **现象**：全局事务提交成功但回滚不生效，脏数据。
- **正确**：接入 Seata 的每个库执行 `seata/undo_log.sql`，上线检查清单必查。

### P-11 多实例下用 `LocalDateTime.now()` 生成业务时间
- **现象**：不同机器时钟差几秒，排序/对账出错。
- **正确**：DB `CURRENT_TIMESTAMP(3)` 或注入统一 `Clock`。

### P-12 导出 Excel 一次性 `SELECT *` 全表
- **现象**：OOM，打挂数据库连接池。
- **正确**：分页流式查询（`ResultSet` 游标）+ EasyExcel 分批写，限制条数。

### P-13 分页页码从 0 开始 vs 从前端 1 开始
- **现象**：前端第 1 页 `page=1`，后端 `Page(0, 10)` 结果是第 2 页。
- **正确**：统一约定 —— **前端 1 基，后端 DTO 收到后 `page - 1`**，在 `PageQuery` 基类里做转换。

### P-14 网关跨域与业务服务跨域同时配置

- **现象**：响应出现两个 `Access-Control-Allow-Origin`。
- **正确**：只在网关配 CORS，业务服务全局禁用它。

## AI 协作

### P-15 AI 编造了不存在的方法
- **正确**：约束 AI 先读源码/类型定义；评审重点看所有 import 是否真实存在。

### P-16 AI 为了过测试而改测试
- **正确**：`CONTRIBUTING.md` 已列为红线；评审时对比测试文件的改动。
