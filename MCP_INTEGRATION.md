# MCP 集成说明（MCP_INTEGRATION）

本文档说明"麦麦点餐助手"实际使用的麦当劳 MCP Server、Tool 清单、调用流程与业务价值。

## 1. 使用的 MCP Server

| 项目 | 说明 |
| --- | --- |
| 服务名称 | 麦当劳中国官方 MCP Server（`mcd-mcp`） |
| 接入地址 | `https://mcp.mcd.cn` |
| 传输协议 | **Streamable HTTP**（JSON-RPC 2.0） |
| 鉴权方式 | 请求头 `Authorization: Bearer <MCP Token>`，Token 在 https://open.mcd.cn 控制台申请 |
| 频率限制 | 每 Token 每分钟 600 次请求，超出返回 HTTP 429（客户端自动等待重试，最多 2 次） |
| 错误码 | 401 Token 无效/过期/未提供；429 触发限流 |
| 协议版本 | 握手协商，客户端声明 `2025-06-18` 并回填服务端返回值，后续请求携带 `MCP-Protocol-Version` 头 |
| 会话管理 | 从 `initialize` 响应头读取 `Mcp-Session-Id` 并在后续请求中回传 |

客户端为本项目**纯 Dart 自研实现**（`lib/mcp/mcp_client.dart`），未依赖任何第三方 MCP SDK，要点：

- `initialize` 握手 → `notifications/initialized` 通知 → `tools/list` / `tools/call`；
- 同时兼容 `application/json` 与 `text/event-stream`（SSE）两种响应形态，SSE 按帧解析 `data:` 并匹配 JSON-RPC id；
- 异常体系：鉴权失败（401）、限流（429，带 Retry-After）、HTTP 错误、JSON-RPC error、工具执行失败（`isError=true`）、网络/超时，均映射为面向用户的中文提示；
- 上述行为有本地假 MCP 服务器的集成测试覆盖（`test/mcp_client_test.dart`，41 个测试全部通过）。

## 2. Tool 清单与使用位置

### 2.1 已接入（应用实际调用）

| Tool | 名称/用途 | 在应用中的位置 | 关键入参 |
| --- | --- | --- | --- |
| `query-nearby-stores` | 查询附近可用门店（到店自取/得来速） | 点餐页 · 门店搜索 | `beType`(1/5)、`searchType=2`、`city`、`keyword` |
| `delivery-query-addresses` | 获取用户可配送地址列表 | 点餐页 · 麦乐送选地址 | — |
| `delivery-create-address` | 新增配送地址 | 点餐页 · 新建地址弹窗 | `city`、`contactName`、`gender`、`phone`、`address`、`addressDetail` |
| `delivery-query-stores` | 查询地址可配送门店 | 点餐页 · 麦乐送 | `addressId`、`beType=2` |
| `query-meals` | 当前门店可售菜单 | 点餐页 · 菜单列表 | `storeCode`、`orderType`(1/2)、`beType`(1/2/5)、`beCode` |
| `query-store-coupons` | 当前门店可用优惠券 | 结算面板 · 券选择 | `storeCode`、`orderType`、`beType`、`beCode` |
| `calculate-price` | 价格计算（含优惠） | 结算面板 · 价格明细；`takeWayList` 亦来源于此 | `items[]`（`productCode`、`quantity`、`couponId/couponCode`）、门店与就餐方式参数 |
| `create-order` | 创建订单（返回订单号与支付链接） | 结算面板 · 创建订单 | `items[]`、`takeWayCode`（到店）或 `addressId`（外送）、`remark`（外送选填） |
| `query-my-account` | 我的积分账户 | 省钱页顶部、我的页 | — |
| `available-coupons` | 麦麦省可领券列表 | 省钱页 · 领券区 | — |
| `auto-bind-coupons` | 麦麦省一键领券 | 省钱页 · 一键领券按钮（带确认弹窗） | — |
| `query-my-coupons` | 我的优惠券列表 | 省钱页 · 我的券 | `page`、`pageSize` |
| `campaign-calendar` | 当月营销活动日历 | 活动页 | `specifiedDate`（选填） |
| `order-list` | 近期历史订单 | 我的页 | — |
| `query-order` | 订单详情 | 我的页 · 订单详情弹窗 | `orderId` |
| `cancel-order` | 取消订单 | 我的页 · 详情内取消（带原因选择） | `orderId`、`cancelReasonCode`（1/2/3/4/5/-1） |

### 2.2 预留未接入（与当前版本边界）

| Tool | 说明 |
| --- | --- |
| `query-meal-detail` | 服务层已封装（`McdService.queryMealDetail`），UI 未接入；用于后续实现套餐"特调/换购"（`modification`/`roundList`） |
| `now-time-info` | 服务层已封装，UI 未调用（当前以设备本地时间代替，用于活动日期选择器初值） |
| `list-nutrition-foods` | 营养查询/控卡搭配，计划作为推荐器的扩展约束 |
| `query-lottery-info` / `draw-lottery` / `query-my-prizes` | 积分抽奖涉及"展示消耗规则并二次确认"的交互链路，暂未接入 |
| `mall-points-products` / `mall-product-detail` / `mall-create-order` / `mall-order-*` | 麦麦商城积分兑换，独立业务线，暂未接入 |
| `query-promotions` / `query-meal-assistance` | 企业团餐（beType=6）专属，当前版本未覆盖团餐场景 |

## 3. 核心调用流程

### 3.0 启动与握手

```
App 启动
  │ 读取配置（优先级：--dart-define > 环境变量 > .env 文件）         ← 本地，无网络
  ▼
McpClient.initialize()
  │ POST initialize {protocolVersion:"2025-06-18", clientInfo}
  │ ← 200 {result:{protocolVersion, serverInfo}} + 响应头 Mcp-Session-Id
  ▼
McpClient.notify("notifications/initialized")   ← 202
  ▼
握手成功 → 进入主页（失败则在配置页展示 401/网络等分类错误原因）
```

### 3.1 到店自取 / 得来速下单（beType=1 或 5，orderType=1）

```
query-nearby-stores {beType, searchType:2, city, keyword}
  → 选定门店（得来速门店携带 beCode）
query-meals {storeCode, orderType:1, beType, beCode?}
  → 菜单渲染 + 本地预算推荐（不调用工具）
  → 用户加购
query-store-coupons {storeCode, orderType:1, beType, beCode?}   ← 结算面板打开时
  → 优惠券选择（默认选中第一张，可取消）
calculate-price {items:[{productCode, quantity, couponId?, couponCode?}], ...}
  → 展示应付金额；读取返回的 takeWayList 供用户选择取餐方式
create-order {items, takeWayCode, ...}
  → 展示 orderId + payUrl（复制到浏览器完成支付）
```

### 3.2 麦乐送下单（beType=2，orderType=2）

```
delivery-query-addresses
  → （列表为空时）delivery-create-address {city, contactName, gender, phone, address, addressDetail}
  → 选定 addressId
delivery-query-stores {addressId, beType:2}
  → 选定门店（携带 beCode）
query-meals → 加购 → query-store-coupons → calculate-price（含配送费展示）
create-order {items, addressId, remark?}
  → 支付链接
```

### 3.3 省钱链路（省钱页）

```
并行加载：query-my-account + available-coupons + query-my-coupons
一键领券（用户确认后）：auto-bind-coupons → 结果提示 → 重新加载上述三项
```

### 3.4 活动与订单链路

```
活动页：campaign-calendar {specifiedDate?}      （指定日期返回该日前後各一有活动日）
我的页：order-list → query-order {orderId} → cancel-order {orderId, cancelReasonCode}
```

## 4. 业务价值

1. **缩短下单路径**：官方 App 的"选店—选餐—结算"多级页面被压缩为单屏"门店 → 菜单 → 结算面板"，从打开应用到完成下单最少 4 次点击；算价与下单走官方 MCP 服务，金额与官方一致。
2. **优惠聚合与自动应用**：领券（麦麦省一键领取）、存量券（我的券）与门店可用券（结算自动带出）三层数据在同一界面呈现；结算时优惠券自动并入 `calculate-price`，用户无需逐券试算。
3. **预算内智能搭配**：基于官方搭配档位指引的本地推荐器（无需额外工具调用），把"选什么"的决策成本降为一次点击，并支持一键入车。
4. **活动可发现**：营销活动集中展示并支持按日期定位，配合领券形成"活动 → 领券 → 下单"的闭环。
5. **对开发者的参考价值**：展示了在无官方 SDK 的语言/平台上如何以纯 Dart 实现 MCP Streamable HTTP 客户端（握手、会话保持、SSE 与 JSON 双响应、限流重试），可作为企业内自研 MCP Client 的模板。

## 5. 调用约定与容错

- **价格单位**：官方返回金额单位为"分"，客户端统一 `fen → ¥x.xx` 展示。
- **字段容错**：部分工具返回字段名未在文档中逐一约定，客户端按"常见命名依次尝试 + 原始 JSON 兜底展示"策略解析，界面提供"原始返回数据"折叠项，保证线上字段变化时应用仍可用、可核对。
- **限流自律**：所有列表页在进入时加载一次、下拉刷新才重拉；429 时客户端按 `Retry-After` 自动退避重试（最多 2 次），未重试成功则向用户明确提示。
- **错误可解释**：401 提示回配置页更换 Token，429 提示稍后重试，网络错误提示检查网络，均直接映射官方错误码语义。
