# 测试规范

## 1. 分层与比例

| 层 | 工具 | 覆盖目标 | 速度要求 |
|---|---|---|---|
| 前端单测 | Vitest + @vue/test-utils | composables / utils / 关键组件逻辑 | 全量 < 30s |
| 前端 E2E | Playwright | 3–5 条核心链路 | 全量 < 5min |
| 后端单测 | JUnit5 + AssertJ + Mockito | Service 分支逻辑 | 单模块 < 60s |
| 后端集成 | @SpringBootTest + Testcontainers | Mapper / 事务 / Feign 契约 | 单模块 < 3min |

**覆盖率不是目标，失败路径才是**。要求：

- Service 层**新增/修改的分支必须被覆盖**（CI 用 JaCoCo 的 changed-lines 检查）。
- 前端 `packages/utils`、`composables` 覆盖率 ≥ 80%；`views` 不强制。
- E2E 只覆盖：登录、主业务流程、权限拦截。

## 2. 命名

```
should_<期望行为>_when_<条件>
例：should_reject_when_stock_insufficient
例：should_return_cached_order_when_redis_hit
```

前端：`describe('OrderService') > it('should reject when stock insufficient')`。

## 3. 前端示例

```ts
// composables/useTable.spec.ts
import { describe, expect, it, vi } from 'vitest'
import { useTable } from './useTable'

describe('useTable', () => {
  it('should_reset_page_when_filter_changes', async () => {
    const fetcher = vi.fn().mockResolvedValue({ list: [], total: 0 })
    const { page, filter, search } = useTable(fetcher, { pageSize: 20 })

    page.value = 3
    filter.value.keyword = 'abc'
    await search()

    expect(fetcher).toHaveBeenLastCalledWith({ page: 1, pageSize: 20, keyword: 'abc' })
  })
})
```

```ts
// e2e/order.spec.ts
import { expect, test } from '@playwright/test'

test('should_create_order_and_see_created_status', async ({ page }) => {
  await page.goto('/login')
  await page.getByTestId('username').fill('qa_order')
  await page.getByTestId('password').fill('123456')
  await page.getByTestId('submit').click()

  await page.goto('/orders/new')
  await page.getByTestId('sku-search').fill('SKU001')
  await page.getByTestId('sku-option-SKU001').click()
  await page.getByTestId('submit-order').click()

  await expect(page.getByTestId('order-status')).toHaveText('已创建')
})
```

## 4. 后端示例

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceImplTest {

    @Mock OrderMapper orderMapper;
    @Mock SkuClient skuClient;
    @InjectMocks OrderServiceImpl orderService;

    @Test
    void should_return_existing_id_when_bizNo_duplicated() {
        Order exist = new Order();
        exist.setId(100L);
        when(orderMapper.selectByBizNo("BIZ-1")).thenReturn(exist);

        Long id = orderService.create(OrderFixture.of("BIZ-1"));

        assertThat(id).isEqualTo(100L);
        verify(orderMapper, never()).insert(any());
    }
}
```

```java
// 集成测试：真实 MySQL，不起 Nacos
@SpringBootTest
@AutoConfigureMockMvc
@Testcontainers
class OrderControllerIT {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
            .withDatabaseName("app").withUsername("app").withPassword("app");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        // 测试里禁用 Nacos：不连注册中心也能跑，避免 CI 需要基础设施
        r.add("spring.datasource.url", mysql::getJdbcUrl);
        r.add("spring.cloud.nacos.discovery.enabled", () -> "false");
        r.add("spring.cloud.nacos.config.enabled", () -> "false");
    }

    @Test
    void should_return_404_when_order_not_found() throws Exception {
        mockMvc.perform(get("/api/v1/orders/999"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.code").value(ErrorCode.ORDER_NOT_FOUND.getCode()));
    }
}
```

## 5. 测试数据

- 统一用 Fixture / Builder 构造，禁止 20 行 `setXxx`。
- 禁止依赖执行顺序、禁止共享可变状态（`@DirtiesContext` 是最后手段）。
- 时间相关逻辑注入 `Clock` Bean，测试用 `Clock.fixed(...)`。
- 前端不 mock `Date` 全局，用 `vi.setSystemTime`。

## 6. 禁止

- ❌ `@Disabled` / `it.skip` 提交（除紧急 hotfix，需带 issue 号且 3 天内修）
- ❌ 为了让测试过而放宽断言、删测试
- ❌ `Thread.sleep`、`cy.wait(3000)` 式的固定等待
- ❌ 快照测试（前端）
- ❌ 在单测里连本地 MySQL / Redis