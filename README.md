# 麦麦点餐助手 (mcd_smart_order)

跨平台智能麦当劳点餐助手，基于麦当劳中国官方 MCP 服务（Streamable HTTP + JSON-RPC 2.0）实现：

- 🍔 **点餐直达下单**：到店自取 / 得来速 / 麦乐送 / **企业团餐** 四种方式；搜索门店 → 浏览菜单 → **点菜单项查看详情**（`query-meal-detail`）→ **选择口味/规格与套餐搭配**（冰量、加浓缩等单选/多选，加价实时计入合计；套餐换购轮次支持换主食/小食/饮品，差价 ±¥x.x 同样实时计算）→ 购物车（可编辑数量/删除/改口味）→ 算价（含优惠券，团餐含助餐服务选择）→ 创建订单并返回支付链接
- 💰 **省钱推荐**：一键领取麦麦省全部可用优惠券（`auto-bind-coupons`）、当前门店可用券（`query-store-coupons`）、积分账户概览（`query-my-account`）
- 🎯 **套餐智能搭配**：按官方档位指引（≤20 元小食；20-30 汉堡+副食；30-40 汉堡+小食+饮料；40-50 丰盛；50+ 丰富组合）给出预算内组合，可一键加入购物车
- 🎉 **活动日历**：当月营销活动查询（`campaign-calendar`），支持按日期定位，按结构化返回（`dailyList[].events[]`）渲染图文活动，「立即抢购」等按钮可查看/复制官方活动页链接
- 🎪 **主题活动预约**：城市 → 门店 → 可预约日期 → 场次 → 预约下单（`query-party-city` / `query-party-store` / `query-party-store-date` / `query-party-store-session` / `party-order-create`）
- 🍎 **餐品营养查询**：常见餐品能量/蛋白/脂肪/碳水/钠/钙（`list-nutrition-foods`，官方为 TOON 紧凑格式，由内置解析器处理）
- 🛒 **麦麦商城**：积分/现金兑换商品，详情含 SKU 规格选择、购买须知与兑换期，实物支持地址配送（`mall-points-products` / `mall-product-detail` / `mall-create-order` / `mall-order-list` / `mall-order-detail`）
- 🎰 **积分抽奖**：活动信息（状态/消耗规则/可用积分）、按资源资格门控的抽奖、我的奖品一览（`query-lottery-info` / `draw-lottery` / `query-my-prizes`）
- 🧾 **订单管理**：历史订单、订单详情、取消订单（`order-list` / `query-order` / `cancel-order`）
- 📱 **响应式布局**：小屏单列；屏幕变宽时**自动增加列数与分栏**（商城商品最多 8 列、活动卡 4 列、门店/菜单 3~4 列、优惠券 3 列），省钱/抽奖/我的页宽屏拆成两栏，点餐页宽屏购物车变成右侧常驻栏——不是把内容缩成窄柱留白。**手机窄屏专项优化**：取餐方式改为可横滑的等宽模式卡（图标在上文字在下，替代会被压扁的四段式分段按钮）、积分统计由三卡横排改为三行纵排、菜单标签截断限 2 个
- 🎨 **个性化**：主题模式（跟随系统/浅色/深色）+ 主题色（品牌红/麦麦黄/活力橙/抹茶绿/青碧/海蓝/魅紫/蜜桃粉 8 套预设，或「自动」跟随系统壁纸动态取色，Android 12+ 生效、不支持的设备回退品牌红）；设置经 SharedPreferences 持久化，全应用即时生效

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
- **`flutter build apk` 报 `SSLHandshakeException: PKIX path building failed`**：Gradle 需要下载 Gradle 发行包，而 `services.gradle.org` 会 307 跳转到 GitHub，本机 JBR 信任库缺少该证书链（OS/curl 正常，仅 Java 失败）。三种解法任选：
  1. **复用本地已有发行包（零下载，最稳）**：wrapper 缓存目录名为 `MD5(distributionUrl)` 的 base36（可用 `md5(URL)` → base36 算出），若本机其它 Gradle home（如 `C:\Users\<user>\.gradle`）已有同版本解压好的发行包，把它整个拷进**生效的** `GRADLE_USER_HOME\wrapper\dists\<name>\<hash>\`（连同 `.ok` 标记文件），并把 `distributionUrl` 改成对应变体即可跳过下载；
  2. **换镜像**：`distributionUrl=https\://mirrors.cloud.tencent.com/gradle/gradle-9.3.1-all.zip`（实测该镜像在 Java 下 TLS 正常）；
  3. **补证书**：用 `keytool -importcert -trustcacerts -cacerts` 把缺失根证书导入 JBR 的 `lib/security/cacerts`（注意 `-Djavax.net.ssl.trustStoreType=Windows-ROOT` 在 JBR 21 已不可用）。
- **Gradle 卡在 `Timeout of 120000 reached waiting for exclusive access to file: ...\\gradle-x.y.z.zip`**：发行包下载被另一个 Gradle 进程持锁（如 Android Studio 或上一次未退干净的构建）。结束残留的 `java.exe`（`gradlew`/daemon）并删除该目录下的 `*.zip.part`、`*.zip.lck` 后重试。
- **注意 `GRADLE_USER_HOME`**：本机被设为 `D:\GradleHome`（非默认的 `%USERPROFILE%\.gradle`），排查 wrapper 缓存时要看这个目录，否则会误判「本地没有缓存」。
- **`flutter build apk` 报 `checkReleaseAarMetadata` 大量 "requires libraries and applications that depend on it to compile against version 32/33/34 or later ... is currently compiled against android-31"**：某些旧插件（如 dynamic_color 1.7.0）在自己的 `android/build.gradle` 里硬编码 `compileSdkVersion 31`，而其 androidx 传递依赖要求更高。修法：在 `android/build.gradle.kts` 的 `subprojects { afterEvaluate { ... } }` 中统一抬升插件子项目 compileSdk（本项目已内置，强制为 36；注意该块必须放在 `evaluationDependsOn(":app")` 之前注册，否则报 "project is already evaluated"）。
- **构建报 `os::commit_memory ... 页面文件太小`（errno 1455）或 `Gradle build daemon disappeared unexpectedly`**：提交内存（物理内存+页面文件）耗尽。本项目 `android/gradle.properties` 已把堆降到 `-Xmx3G` 并限制 `org.gradle.workers.max=2`、`kotlin.compiler.execution.strategy=in-process`；构建前可先 `gradlew --stop` 清理残留守护进程，并关闭占内存的大程序（8GB 物理内存的机器尤为明显）。
- **返回字段与预期不符**：工具返回中的字段名未全部在文档约定，应用对常见命名做了容错解析，并在各页面提供"原始返回数据"折叠视图便于核对。

## 使用示例

以下均为应用内真实操作路径（括号内为对应的 MCP 工具）：

### 示例 1：到店自取，30 元预算智能搭配

1. 启动应用 → 在配置页粘贴 MCP Token → 连接成功进入主页
2. 「点餐」页选择 **到店自取**（窄屏为可横滑的模式卡），输入城市（如"上海市"）与位置关键词 → 搜索门店（`query-nearby-stores`）
3. 点击门店进入菜单（`query-meals`），点击 **¥30 智能搭配**，应用按官方档位给出组合，例如"吉士汉堡 + 麦乐鸡 5 块"（本地推荐算法，无需调用工具）
4. 点击 **一键加入** → 底部购物车显示件数与合计 → **去结算**
5. （可选）点任意菜单项进入详情，选择口味后 **加入购物车**（如冰美式选"去冰"；加价项如"加浓缩 +¥5.00"实时计入合计，同商品不同口味在购物车中分行显示，可随时"改口味"）
6. 结算面板自动加载当前门店可用优惠券（`query-store-coupons`）并算价（`calculate-price`），显示"商品合计 / 优惠 / 应付"与取餐方式（来自算价结果的 `takeWayList`）
7. 点击 **创建订单**（`create-order`）→ 显示订单号与支付链接，复制到浏览器即可完成支付

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
├── main.dart                  # 入口：配置加载 → 连接 MCP → 主页（DynamicColorBuilder + 主题应用）
├── app/session.dart           # MCP 会话（initialize 握手 + 服务封装）
├── app/theme.dart             # 个性化：主题模式/主题色预设/ThemeController（SharedPreferences 持久化）
├── config/app_config.dart     # Token/URL 配置（dart-define → 环境变量 → .env）
├── mcp/mcp_client.dart        # MCP 客户端：Streamable HTTP + JSON-RPC 2.0，
│                              #   支持 application/json 与 text/event-stream 两种响应，
│                              #   Mcp-Session-Id 会话保持、Bearer 鉴权、429 自动重试
├── services/
│   ├── mcd_service.dart       # 麦当劳工具类型化封装（query-meals / calculate-price / create-order …）
│   ├── menu_parser.dart       # 工具返回结果的容错解析（门店/菜单/优惠券/地址/营养）
│   ├── models.dart            # MenuItemInfo / CartLine / StoreInfo / CouponInfo / NutritionInfo
│   └── recommender.dart       # 预算套餐推荐（纯 Dart，可单测）
├── utils/
│   └── toon.dart              # TOON 紧凑格式解析（list-nutrition-foods 的 data）
└── ui/
    ├── setup_page.dart        # Token 配置页
    ├── order_tab.dart         # 点餐：选店（到店/得来速/麦乐送/团餐）→ 菜单 → 餐品详情 → 购物车 → 结算下单
    ├── save_tab.dart          # 省钱：一键领券 / 积分 / 我的券 / 餐品营养
    ├── events_tab.dart        # 活动日历 + 主题活动预约
    ├── party_booking.dart     # 主题活动预约（城市→门店→日期→场次→下单）
    ├── mall_tab.dart          # 麦麦商城：商品列表 / 详情 / 兑换下单 / 订单
    ├── lottery_tab.dart       # 积分抽奖：活动信息 / 抽奖 / 我的奖品
    ├── me_tab.dart            # 我的：历史订单 / 取消订单 / 服务器时间 / 个性化（主题模式+主题色）
    └── widgets.dart           # 通用组件与金额格式化（价格单位：分）
```

> 依赖：flutter / cupertino_icons / flutter_lints + shared_preferences（主题设置持久化）+ dynamic_color（系统动态取色，Android 12+ 生效）；开发期工具 flutter_launcher_icons。
>
> 应用图标：工程根目录 `logo.png`（1080×1080）。修改图标后运行 `dart run flutter_launcher_icons` 重新生成 Android / iOS / Windows / macOS / Web 各平台图标（配置见 pubspec.yaml 的 `flutter_launcher_icons` 段）。

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

- 单品**口味/规格特调**（`modification`）与**套餐换购轮次**（`roundList`）均已支持：详情弹层按 `minValues`/`maxValues` 渲染单选/多选、加价与份数，套餐轮次支持更换主食/小食/饮品（含差价与子项特调），按官方格式随下单提交（`items[].modification.values[]` 每项 `{code, quantity, key}`；`items[].roundList[]` 每项 `{round, comboItemList: [{code, quantity, modification?}]}`），实测算价接口接受、差价按分精确生效。
- 企业团餐（beType=6）已支持（含助餐服务选择并以 `gmServiceCode` 提交 `create-order`）；预约单（预约到店）暂未覆盖。
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
