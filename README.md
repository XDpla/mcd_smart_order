# 麦麦点餐助手 (mcd_smart_order)

跨平台智能麦当劳点餐助手，基于麦当劳中国官方 MCP 服务（Streamable HTTP + JSON-RPC 2.0）实现：

- 🍔 **点餐直达下单**：到店自取 / 得来速 / 麦乐送三种方式；搜索门店 → 浏览菜单 → 购物车 → 算价（含优惠券）→ 创建订单并返回支付链接
- 💰 **省钱推荐**：一键领取麦麦省全部可用优惠券（`auto-bind-coupons`）、当前门店可用券（`query-store-coupons`）、积分账户概览（`query-my-account`）
- 🎯 **套餐智能搭配**：按官方档位指引（≤20 元小食；20-30 汉堡+副食；30-40 汉堡+小食+饮料；40-50 丰盛；50+ 丰富组合）给出预算内组合，可一键加入购物车
- 🎉 **活动日历**：当月营销活动查询（`campaign-calendar`），支持按日期定位
- 🧾 **订单管理**：历史订单、订单详情、取消订单（`order-list` / `query-order` / `cancel-order`）

> 平台：Windows / Android（Flutter 3.47+，Dart 3.13+）。
> 相关文档：[MCP 集成说明](MCP_INTEGRATION.md) · [参赛声明](CONTEST_DECLARATION.md) · [开发上下文](workbuddy.md)

## 目标用户

| 用户群体 | 场景与价值 |
| --- | --- |
| **麦当劳会员 / 常点用户** | 一个界面完成"搜门店 → 看菜单 → 用券 → 下单支付"全流程，无需在多个 App 页面间跳转；订单号与支付链接直达 |
| **追求优惠的"省钱党"** | 省钱页聚合积分、可领券与已领券，支持一键领取麦麦省全部优惠券，结算时自动带出当前门店可用券并算价对比 |
| **选择困难 / 预算敏感用户** | 输入预算（¥20/30/40/50）即按官方搭配档位生成套餐组合，一键加入购物车，减少决策成本 |
| **关注活动的用户** | 活动日历集中展示当月营销活动，支持按日期定位前后活动 |
| **AI / MCP 应用开发者** | 纯 Dart 实现的 MCP Streamable HTTP 客户端（含握手、会话保持、SSE 解析、429 重试）可作为自研 MCP Client 的参考实现 |

## 安装方法

### 1. 环境准备

- Flutter 3.47+（含 Dart 3.13+），Windows 桌面构建需 Visual Studio 2022 桌面开发负载，Android 构建需 Android SDK
- 一枚有效的麦当劳 MCP Token（见下）

### 2. 申请 MCP Token

1. 访问 [https://open.mcd.cn](https://open.mcd.cn) 并登录
2. 点击右上角 **控制台** → **激活** 并申请 MCP Token
3. Token 与你的麦当劳会员账户绑定，**请妥善保管，避免泄露**

### 3. 配置 Token（三选一，按优先级从高到低）

| 方式 | 适用场景 | 做法 |
| --- | --- | --- |
| `--dart-define` | 移动端 / 正式构建 | `flutter run --dart-define=MCD_MCP_TOKEN=你的Token` |
| 环境变量 | 桌面调试 | `set MCD_MCP_TOKEN=你的Token`（Windows）或 `export ...` |
| `.env` 文件 | 桌面端 | 复制 `.env.example` 为 `.env` 并填入 Token |

`.env` 示例（占位符，替换为真实值；**该文件已被 .gitignore 忽略，不会进入仓库**）：

```dotenv
MCD_MCP_TOKEN=YOUR_MCP_TOKEN
MCD_MCP_URL=https://mcp.mcd.cn
```

> ⚠️ **安全约定**：严禁将真实 Token 硬编码到源码或提交到 Git 仓库。首次运行时也可以在应用内配置页直接粘贴 Token（仅保存在内存；桌面端可勾选"记住到 .env"）。

### 4. 获取与运行

```bash
git clone <本仓库地址>
cd mcd_smart_order
flutter pub get
flutter run -d windows   # Windows 桌面端
flutter run -d <device>  # Android 设备/模拟器
```

测试与静态检查：

```bash
flutter analyze
flutter test
```

### 5. 常见问题（Troubleshooting）

- **`flutter` 命令报 `Unable to 'pub upgrade' flutter tool` / 引导失败**
  Flutter 引导依赖 git。若系统未安装 git，可将 Flutter SDK 自带的 mingit 加入 PATH 后重试：
  `set PATH=D:\flutter\bin\mingit\cmd;%PATH%`（或安装任意版 git）。本项目开发时即遇到并如此解决。
- **HTTP 401**：Token 无效/过期/未携带，请检查 `Authorization: Bearer <token>` 配置。
- **HTTP 429**：触发限流（每 Token 600 次/分钟），客户端已自动重试，仍失败请降低操作频率。
- **返回字段与预期不符**：工具返回中的字段名未全部在文档约定，应用对常见命名做了容错解析，并在各页面提供"原始返回数据"折叠视图便于核对。

## 使用示例

以下均为应用内真实操作路径（括号内为对应的 MCP 工具）：

### 示例 1：到店自取，30 元预算智能搭配

1. 启动应用 → 在配置页粘贴 MCP Token → 连接成功进入主页
2. 「点餐」页选择 **到店自取**，输入城市（如"上海市"）与位置关键词 → 搜索门店（`query-nearby-stores`）
3. 点击门店进入菜单（`query-meals`），点击 **¥30 智能搭配**，应用按官方档位给出组合，例如"吉士汉堡 + 麦乐鸡 5 块"（本地推荐算法，无需调用工具）
4. 点击 **一键加入** → 底部购物车显示件数与合计 → **去结算**
5. 结算面板自动加载当前门店可用优惠券（`query-store-coupons`）并算价（`calculate-price`），显示"商品合计 / 优惠 / 应付"与取餐方式（来自算价结果的 `takeWayList`）
6. 点击 **创建订单**（`create-order`）→ 显示订单号与支付链接，复制到浏览器即可完成支付

### 示例 2：麦乐送到家 + 新建地址

1. 「点餐」页选择 **麦乐送** → 首次使用点击 **选择** → 填写城市、联系人、称呼、手机号、地址与门牌号（`delivery-create-address`）
2. 选定地址后自动查询可配送门店（`delivery-query-stores`）→ 选店进菜单
3. 结算时可填写备注（如"无接触配送"，随 `create-order` 提交），配送费与应付金额由算价结果（`calculate-price`）给出

### 示例 3：一键领券 + 查看积分

1. 进入「省钱」页 → 顶部卡片显示可用 / 即将过期 / 累计积分（`query-my-account`）
2. 「麦麦省 · 可领优惠券」列表（`available-coupons`）→ 点击 **一键领券** 并确认（`auto-bind-coupons`）→ 提示领取结果并自动刷新
3. 下方"我的优惠券"展示已领到的券（`query-my-coupons`）

### 示例 4：查活动、管理订单

1. 「活动」页查看当月营销活动（`campaign-calendar`），点击右上角日历图标可按指定日期定位前后活动
2. 「我的」页查看历史订单（`order-list`），点击订单查看详情（`query-order`），未支付订单可在详情内选择原因取消（`cancel-order`）

### 示例 5：命令行方式配置 Token 运行（适合演示/评测）

```bash
flutter run -d windows --dart-define=MCD_MCP_TOKEN=你的Token
```

## 架构

```
lib/
├── main.dart                  # 入口：配置加载 → 连接 MCP → 主页
├── app/session.dart           # MCP 会话（initialize 握手 + 服务封装）
├── config/app_config.dart     # Token/URL 配置（dart-define → 环境变量 → .env）
├── mcp/mcp_client.dart        # MCP 客户端：Streamable HTTP + JSON-RPC 2.0，
│                              #   支持 application/json 与 text/event-stream 两种响应，
│                              #   Mcp-Session-Id 会话保持、Bearer 鉴权、429 自动重试
├── services/
│   ├── mcd_service.dart       # 麦当劳工具类型化封装（query-meals / calculate-price / create-order …）
│   ├── menu_parser.dart       # 工具返回结果的容错解析（门店/菜单/优惠券/地址）
│   ├── models.dart            # MenuItemInfo / CartLine / StoreInfo / CouponInfo
│   └── recommender.dart       # 预算套餐推荐（纯 Dart，可单测）
└── ui/
    ├── setup_page.dart        # Token 配置页
    ├── order_tab.dart         # 点餐：选店 → 菜单 → 购物车 → 结算下单
    ├── save_tab.dart          # 省钱：一键领券 / 积分 / 我的券
    ├── events_tab.dart        # 活动日历
    ├── me_tab.dart            # 我的：历史订单 / 取消订单
    └── widgets.dart           # 通用组件与金额格式化（价格单位：分）
```

MCP 服务器接入配置（等价 JSON，供其它 MCP Client 参考）：

```json
{
  "mcpServers": {
    "mcd-mcp": {
      "type": "streamablehttp",
      "url": "https://mcp.mcd.cn",
      "headers": {
        "Authorization": "Bearer YOUR_MCP_TOKEN"
      }
    }
  }
}
```

## 已知边界

- 套餐"特调/换购"（`modification` / `roundList`）暂未支持，当前按默认组成整单下单；价格与库存以门店算价结果为准。
- 企业团餐（beType=6）、预约单暂未覆盖。
- 支付通过 `create-order` 返回的支付链接完成（应用内提供复制），不内置支付 SDK。
- 工具返回字段命名以线上实际为准，解析器采取"常见命名尝试 + 原始数据兜底"策略。

## 安全清单

- [x] 源码中无任何硬编码 Token（全部经配置注入）
- [x] `.env`、`.env.*`、`*.env` 已加入 `.gitignore`，`.env.example` 仅含占位符
- [x] README 提供配置说明与泄露风险提示
- [x] Android `INTERNET` 权限已声明（release 包可正常联网）

## 相关链接

- MCP 集成说明（Server / Tool / 调用流程 / 业务价值）：[MCP_INTEGRATION.md](MCP_INTEGRATION.md)
- 参赛声明（原创性 / 合规性 / 敏感信息）：[CONTEST_DECLARATION.md](CONTEST_DECLARATION.md)
- 开发对话上下文（WorkBuddy 联动核验）：[workbuddy.md](workbuddy.md)
- 麦当劳 MCP 文档：https://open.mcd.cn/mcp/doc
- MCP 协议：https://modelcontextprotocol.io
