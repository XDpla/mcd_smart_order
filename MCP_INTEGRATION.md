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
- 上述行为有本地假 MCP 服务器的集成测试覆盖（`test/mcp_client_test.dart` 等，共 **126 个测试全部通过**，含 41 个客户端集成测试与解析/模型/配置/组件测试，覆盖商城、抽奖、团餐助餐、活动日历结构化解析、业务错误识别、错误页返回、响应式布局、购物车编辑、口味特调与套餐换购提交、主题个性化与窄屏布局等）。

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
| `calculate-price` | 价格计算（含优惠） | 结算面板 · 价格明细；`takeWayList` 亦来源于此 | `items[]`（`productCode`、`quantity`、`couponId/couponCode`、`modification?`）、门店与就餐方式参数 |
| `create-order` | 创建订单（返回订单号与支付链接） | 结算面板 · 创建订单 | `items[]`（含 `modification`）、`takeWayCode`（到店）或 `addressId`（外送）、`remark`（外送选填） |
| `query-my-account` | 我的积分账户 | 省钱页顶部、我的页 | — |
| `available-coupons` | 麦麦省可领券列表 | 省钱页 · 领券区 | — |
| `auto-bind-coupons` | 麦麦省一键领券 | 省钱页 · 一键领券按钮（带确认弹窗） | — |
| `query-my-coupons` | 我的优惠券列表 | 省钱页 · 我的券 | 无需入参（官方文档） |
| `campaign-calendar` | 当月营销活动日历 | 活动页 | `specifiedDate`（选填）；真实返回含 `structuredContent`：`data.dailyList[].events[]`（标题/内容/图片在 `articleDto`），Markdown 文本仅作回退 |
| `list-nutrition-foods` | 餐品营养信息列表 | 省钱页 · 餐品营养面板 | 无需入参；`data` 为 **TOON 字符串**，需 `Toon.parseTable` 解析 |
| `query-party-city` | 主题活动可参与城市列表 | 活动页 · 主题活动预约 | `spuId` |
| `query-party-store` | 主题活动可参与门店列表 | 活动页 · 主题活动预约 | `code`（城市 code，必填）、`spuId`、`latitude`/`longitude`（选填，来自城市列表） |
| `query-party-store-date` | 主题活动可预约日期 | 活动页 · 主题活动预约 | `spuId`、`storeCode` |
| `query-party-store-session` | 主题活动可预约场次 | 活动页 · 主题活动预约 | `spuId`、`storeCode`、`dateStr` |
| `party-order-create` | 主题活动下单（返回订单号与 `payH5Url`） | 活动页 · 主题活动预约 | `partyType`（1-包场/2-拼团，必填）、`spuId`、`skuId`、`storeCode`、`dateStr`、`code`、`id`、`timeStart`、`timeEnd`、`leftNum`、`count` |
| `order-list` | 近期历史订单 | 我的页 | — |
| `query-order` | 订单详情 | 我的页 · 订单详情弹窗 | `orderId` |
| `cancel-order` | 取消订单 | 我的页 · 详情内取消（带原因选择） | `orderId`、`cancelReasonCode`（1/2/3/4/5/-1） |
| `query-meal-detail` | 餐品详情（图文/标签/描述/口味/搭配） | 点餐页 · 点菜单项弹出详情（含口味与换购选择） | `storeCode`、`orderType`、`beType`、`code`、`beCode?`；`modification.items[].values[]` 为口味/规格选项（`code`/`price`(分)/`name`/`maxQuantity`/`selectedQuantity`/`selectedKey`），`rounds[].choices[]` 为套餐换购子项（`diffPrice` 带符号差价、`supportModify` 子项特调） |
| `now-time-info` | 服务器当前时间 | 我的页 · 服务器时间 | 无需入参 |
| `query-meal-assistance` | 企业团餐助餐服务查询 | 点餐页 · 团餐结算面板（必选助餐） | `storeCode`、`beCode`、`beType=6`、`orderType=2` |
| `mall-points-products` | 麦麦商城积分/现金商品列表 | 商城页 · 商品网格 | `catRuleIds`（选填）；字段为 `spuName/spuId/point(积分)/price(元字符串)/spuImage/catName/selling/status` |
| `mall-product-detail` | 商城商品详情 | 商城页 · 商品详情弹窗 | `spuId`；积分/价格在 `skuList[].{skuId,points,price,extTradePrice,specList}`，图片在 `images[]`，文案 `detail`/`note` 为 HTML（去标签展示），另有 `upDate/downDate`、`shopId`、派对信息与 `spuLimit` |
| `mall-create-order` | 商城积分兑换下单 | 商城页 · 商品详情弹窗下单 | `skuId`（来自详情 `skuList`，**不是 spuId**）、`spuCategory`('1'虚拟/'2'实物)、`count?`、`addressId?`(实物必填)；仅支持 `shopId=2`，派对/活动类须走 `party-order-create` |
| `mall-order-list` | 商城近一年订单 | 商城页 · 订单列表 | `lastId?`、`size?` |
| `mall-order-detail` | 商城订单详情 | 商城页 · 订单详情 | `orderId` |
| `query-lottery-info` | 积分抽奖活动信息（状态/次数/奖池） | 抽奖页 · 顶部状态 | 无需入参；状态文案在 `activityStatusText`（`status` 是 `{code,message}` 对象），资格以 `drawDecision.resourceEligible` 为准、消耗以 `drawDecision.nextConsumption` 为准 |
| `draw-lottery` | 执行一次抽奖（返回中奖结果与剩余次数） | 抽奖页 · 抽奖按钮 | 无需入参 |
| `query-my-prizes` | 我的抽奖奖品记录 | 抽奖页 · 奖品列表 | `pageNum?`、`pageSize?` |

### 2.2 覆盖说明

官方文档（`https://open.mcd.cn/mcp/doc`）共描述 **33 个 MCP 工具**，上表（2.1）已全部接入并在对应页签提供 UI 入口；`query-promotions` 未出现在线上 `tools/list` 的 33 工具清单中，故未实现。剩余可深化的方向（非阻塞、不依赖新增 MCP 工具）：商城实物兑换时的"随单购"（`withOrder`）。单品口味特调（`modification`）与套餐换购轮次（`roundList`）均已支持，见 §3.6 与 §5。

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
  → 用户加购（默认口味直加；需要口味时点开餐品详情，见 §3.6）
query-store-coupons {storeCode, orderType:1, beType, beCode?}   ← 结算面板打开时
  → 优惠券选择（默认选中第一张，可取消）
calculate-price {items:[{productCode, quantity, couponId?, couponCode?, modification?}], ...}
  → 展示应付金额；读取返回的 takeWayList 供用户选择取餐方式
create-order {items, takeWayCode, ...}
  → 展示 orderId + payUrl（复制到浏览器完成支付）
```

### 3.6 口味特调与套餐换购（modification / roundList）链路

```
用户点菜单项 → query-meal-detail {storeCode, orderType, beType, code, beCode?}
  → data.modification.items[] 分组渲染（单品口味）：
      · maxValues=1 → 单选（互斥，如 标准/去冰/多冰/少冰）
      · maxValues>1 → 多选（如配料，最多 N 项）
      · minValues>=1 → 必选（未选时禁止加入购物车）
      · value.price>0 → 加价项（实时计入合计；maxQuantity>1 时提供份数步进）
      · value.selectedQuantity>0 → 服务端默认选中（如汉堡默认配料）
  → data.rounds[] 分组渲染（套餐换购）：
      · 每轮 {id, name, minQuantity, maxQuantity, choices[]}
      · choices[] 为可选子项，diffPrice 为带符号差价（"+ ¥2.5" / "- ¥0.5"）
      · choice.quantity>0 → 默认选中（isDefault 仅提示，实测有 isDefault=1 但 quantity=0）
      · choice.supportModify → 子项自身可特调（嵌套 modification，结构同上）
  → 加入购物车（同商品不同口味/搭配分行；可「改口味」更新原行）
calculate-price / create-order 的 items[] 元素：
  单品口味：{productCode, quantity, modification: {values: [{code, quantity, key}]}}
    · key 取该选项的 selectedKey **原样透传**（线上不遵循序号规律，禁止拼接）
    · 仅提交「已选中」的选项（0 份不提交）
  套餐换购：{productCode, quantity, roundList: [{round: 轮次id,
              comboItemList: [{code, quantity, modification?}]}]}
    · round 取详情 rounds[].id（非法 id 服务端报 60006）
    · 子项特调结构与 modification 相同（提交到 comboItemList[].modification）
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

### 3.5 企业团餐下单（beType=6，orderType=2）

```
delivery-query-addresses →（空则）delivery-create-address → 选定 addressId
delivery-query-stores {addressId, beType:6}          ← 团餐复用外送门店查询
  → 选定门店（携带 beCode）
query-meals {storeCode, orderType:2, beType:6, beCode}
  → 加购 → query-store-coupons → calculate-price
query-meal-assistance {storeCode, beCode, beType:6, orderType:2}
  → 结算面板列出可选助餐服务（保鲜速达/专人分餐等，含默认选中项与促销说明），用户必选其一
create-order {items, addressId, gmServiceCode, remark?}
  → gmServiceCode 取自 query-meal-assistance 的 mealAssistanceItems[].gmServiceCode
  → 展示 orderId + payUrl
```

> 团餐（beType=6）与麦乐送（beType=2）同属 `orderType=2`，故门店发现、地址、备注、配送费等逻辑共用；差异仅在：结算面板需额外选择"助餐服务"并以 `gmServiceCode` 提交 `create-order`。


## 4. 业务价值

1. **缩短下单路径**：官方 App 的"选店—选餐—结算"多级页面被压缩为单屏"门店 → 菜单 → 结算面板"，从打开应用到完成下单最少 4 次点击；算价与下单走官方 MCP 服务，金额与官方一致。
2. **优惠聚合与自动应用**：领券（麦麦省一键领取）、存量券（我的券）与门店可用券（结算自动带出）三层数据在同一界面呈现；结算时优惠券自动并入 `calculate-price`，用户无需逐券试算。
3. **预算内智能搭配**：基于官方搭配档位指引的本地推荐器（无需额外工具调用），把"选什么"的决策成本降为一次点击，并支持一键入车。
4. **活动可发现**：营销活动集中展示并支持按日期定位，配合领券形成"活动 → 领券 → 下单"的闭环。
5. **对开发者的参考价值**：展示了在无官方 SDK 的语言/平台上如何以纯 Dart 实现 MCP Streamable HTTP 客户端（握手、会话保持、SSE 与 JSON 双响应、限流重试），可作为企业内自研 MCP Client 的模板。

## 5. 调用约定与容错

- **入参以线上 `tools/list` 的 JSON Schema 为准**（网页文档存在笔误/不一致，务必以 schema 校验）：例如主题活动工具真名为 `query-party-store-date` / `query-party-store-session`（非文档正文的 `query-partystore-*`），建址城市参数为 `city`（非 `name`）。所有工具 schema 均为 `additionalProperties: false`，**多传任何未声明字段都会被服务端拒绝**，因此客户端只发送 schema 中声明的参数。关键规则：
  - `beType`：1-到店自取，2-麦乐送，5-得来速(DT)，6-企业团餐；`orderType`：1-到店（含到店自取+得来速），2-外送（含麦乐送+团餐）；
  - `beCode`：到店自取(beType=1) **不传**（传了会报错）；得来速(5)/外送(2)/团餐(6) **必传**（来自 `query-nearby-stores` / `delivery-query-stores`）；
  - `takeWayCode`：仅到店(orderType=1)必传，取自 `calculate-price` 返回的 `data.takeWayList[].code`；`addressId`：仅外送(orderType=2)必填；`gmServiceCode`：仅企业团餐(beType=6)必传，取自 `query-meal-assistance` 的 `mealAssistanceItems[].gmServiceCode`；
  - 优惠券只能通过 `items[].couponId` / `items[].couponCode` 传递（schema 无顶层同名字段）。
- **响应形态不统一**（同一套工具返回三种形态，均已兼容）：
  - **JSON**：`query-nearby-stores` / `query-meals` / `query-my-account` / 主题活动各工具等；
  - **Markdown 文本**：`campaign-calendar` / `available-coupons` / `query-my-coupons` / `auto-bind-coupons` —— 客户端在 `payloadFromToolResult` 中回退为 `{'raw': '<markdown>'}`，界面直接渲染 Markdown，不做结构化解析；
  - **TOON 字符串**：`list-nutrition-foods` 的 `data` 为 TOON 表格字符串（`[N]{列名,...}:` + 数据行），由 `lib/utils/toon.dart` 解析。
- **菜单结构为「分类引用 + 详情表」**：`query-meals` 的 `data.categories[].meals` 仅含 `code`/`tags`，名称/价格/图片在 `data.meals`（按 code 索引的映射）中，解析时需回查合并。
- **口味（modification）只在详情接口返回**（2026-10-25 实测）：`query-meals` 全网扫描不含 `modification` 字段，唯一入口是 `query-meal-detail`；且详情接口**可能不返回价格**（如 507411 仅有 `code/name/rounds/image/modification/supportModify`），此时价格回退菜单列表价。提交要点：
  - 结构为 `items[].modification.values[]` 每项 `{code, quantity, key}`（`calculate-price` 与 `create-order` 一致）；
  - `key` 用选项的 `selectedKey` **原样透传**——线上数据不遵循"组号-序号"规律（冰美式「少冰」实测为 `1-1`），自行拼接会被服务端拒绝；
  - `unselectedKey` 文档示例中有、线上实测缺失，代码按可空处理；
  - 实测「仅提交选中值（selectedKey）」与「全量值（选中/未选 key）」两种形态均被 `calculate-price` 接受，客户端采用前者（更精简）；
  - `values` 为空的组（如线上美式的「浓缩」组）在解析阶段剔除；
  - 同组内多个选项 `code` 可能相同（冰量四项均为 `200002`），选项身份以 `selectedKey`（缺失时 `code#name`）区分；
  - 加价 `price` 为 int、单位**分**（如加浓缩 500 = ¥5.00），与顶层价格的元字符串约定不同。
- **套餐换购（roundList）同样只在详情接口返回**（2026-10-25 实测）：`rounds[].{id,name,minQuantity,maxQuantity,choices[]}`，子项 `{code,name,quantity,maxQuantity,isDefault,diffPrice,supportModify,modification}`：
  - 提交格式 `items[].roundList[]` 每项 `{round: 轮次id, comboItemList: [{code, quantity, modification?}]}`，与 calculate-price / create-order 一致；
  - `diffPrice` 为带符号元字符串（"+ ¥2.5" / "- ¥0.5"），解析为含符号的分，实测算价精确生效（2790→3040 / 2740）；
  - 默认选中以 `choice.quantity>0` 为准（实测有 `isDefault=1` 但 `quantity=0` 的情况，不可轻信 isDefault）；
  - 子项 `supportModify=true` 时带嵌套 `modification`，提交到 `comboItemList[].modification`（结构与单品特调相同）；
  - 非法 `round` id 服务端报 60006「商品已售罄或不可售」，故轮次 id 必须取自详情返回。
- **价格单位**：菜单 `currentPrice`/`originalPrice` 为**元字符串**（如 `"33.5"`、`"30"`）；算价与订单结果（`payableTotal` 等）为**分**。客户端分别按"字符串/小数按元、整数按分"与"分"处理，统一 `fen → ¥x.xx` 展示。
- **字段容错**：部分工具返回字段名未在文档中逐一约定，客户端按"常见命名依次尝试 + 原始 JSON 兜底展示"策略解析，界面提供"原始返回数据"折叠项，保证线上字段变化时应用仍可用、可核对。
- **限流自律**：所有列表页在进入时加载一次、下拉刷新才重拉；429 时客户端按 `Retry-After` 自动退避重试（最多 2 次），未重试成功则向用户明确提示。
- **业务错误体直显**：部分工具把 `{success:false, code, message}` 作为**正常结果**（非 `isError`）返回（如 600057「门店可能已关闭或不在营业时间」、600048「beType=2/5/6 时必传 beCode」）。客户端在 `McpClient`（isError 文本）与 `McdService._call`（正常 payload）两层检测该结构，提取 `message` 与 `code` 直接抛出 `McpToolException`，由对应页签以错误卡/弹窗展示**真实报错原因**（而非吞成空态或泛化错误），用户据此即可定位（如切换地址/门店、检查营业时间）。这一层有回归测试覆盖（`test/parsers_test.dart` 的"业务错误识别"组）。
- **错误可解释**：401 提示回配置页更换 Token，429 提示稍后重试，网络错误提示检查网络，业务错误体（见上）展示服务端 `message`（含错误码），均直接映射官方语义。
