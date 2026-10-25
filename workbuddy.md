# WorkBuddy 开发上下文（workbuddy.md）

本文件用于核验本项目是否符合 WorkBuddy 联动活动奖励条件，记录项目与 WorkBuddy 协作开发的过程上下文。

## 1. 项目与 WorkBuddy 的关系

本项目由作者使用 WorkBuddy（AI 应用构建工具）发起：作者向 WorkBuddy 输入下文第 2 节的提示词，WorkBuddy 生成了项目的初始工程骨架。**WorkBuddy 阶段的产出为 Flutter 默认工程模板**（详见第 3 节核验记录）；随后作者在同一项目上继续迭代，完成了 MCP 客户端与全部业务功能的实现（详见第 4 节）。

## 2. 作者输入 WorkBuddy 的提示词（原文）

> 请参考 https://open.mcd.cn/mcp/doc 中的 MCP 协议，设计并实现一个跨平台智能麦当劳点餐助手,为用户提供省钱,活动,套餐的推荐,并可直达下单
> 由于项目将上传至 GitHub，必须严格保障安全性：严禁将 MCP Token 硬编码或提交至仓库；请使用环境变量或本地配置文件（如 .env）管理密钥，并确保配置文件已被 .gitignore 忽略；同时在 README 中提供配置说明与占位符示例，确保公开仓库中不存在泄露风险。

## 3. WorkBuddy 阶段产出核验（2026-10-24 检查记录）

对 WorkBuddy 生成结果进行检查时的实际仓库状态（即 WorkBuddy **初始产出快照**，用于核验其仅提供工程骨架；仓库后续已由作者完整实现，详见第 4 节）：

- `lib/main.dart`：Flutter `flutter create` 默认计数器示例（未包含任何麦当劳 / MCP 相关代码）；
- `test/widget_test.dart`：默认计数器组件测试；
- `README.md`：默认模板文本（"A new Flutter project."），未包含 MCP 配置说明；
- `.gitignore`：Flutter 默认模板，**未包含 `.env` 等密钥文件忽略规则**；
- `pubspec.yaml`：默认依赖（仅 flutter / cupertino_icons / flutter_lints），无任何 MCP 相关依赖；
- 已生成 android/ 与 windows/ 平台目录，`.metadata` 显示模板基于 Flutter stable `abaf9c52…` 生成。

即：WorkBuddy 完成了"可运行的 Flutter 工程骨架"，但原始需求中的 MCP 协议接入、点餐/省钱/活动功能与安全配置均未落地。本快照仅反映 WorkBuddy 输出那一刻的状态；当前仓库（经第 4 节迭代后）已变为完整实现。

## 4. 后续迭代记录（2026-10-24 整理，与最终提交内容一致）

在 WorkBuddy 骨架基础上，作者通过 AI 编码助手会话完成以下实现（已与当前仓库最终状态核对一致）：

1. **修复 Flutter 工具链**：定位到本机 PATH 缺 git 导致 `flutter.bat` 引导失败，改用 SDK 自带 mingit 恢复 `flutter pub get / analyze / test`（已写入 README Troubleshooting）；
2. **MCP 客户端**：`lib/mcp/mcp_client.dart` — 纯 Dart 实现 Streamable HTTP + JSON-RPC 2.0；`initialize` 握手并读取 `Mcp-Session-Id` 会话保持、回传 `MCP-Protocol-Version` 协商版本；同时兼容 `application/json` 与 `text/event-stream`（SSE）双响应；异常体系覆盖 401/403 鉴权、429 限流（按 `Retry-After` 退避、最多 2 次重试）、HTTP 错误、JSON-RPC 错误、工具 `isError`、网络/超时；服务层（`lib/services/mcd_service.dart`）优先解析 `structuredContent`，否则解析首个 text content 中的 JSON，失败回退 `raw`，保证线上字段变化仍可用；
3. **安全配置**：`lib/config/app_config.dart` — Token 经 `--dart-define` / 环境变量 / `.env` 三级注入（桌面端可写回 `.env`，移动端用 dart-define）；`.gitignore` 补 `.env`、`.env.*`、`*.env` 并保留 `!.env.example`；新增 `.env.example` 占位符；README 补配置与安全说明；Android 主清单声明 `INTERNET` 权限（release 包联网必需）；
4. **业务功能**：点餐（到店自取 `beType=1` / 得来速 `beType=5` / 麦乐送 `beType=2` / **企业团餐 `beType=6`** 四种方式，搜店→菜单→**点菜单项查看详情**→购物车→算价（含门店券，团餐含助餐服务选择）→创建订单并返回支付链接，直达下单）；省钱（麦麦省一键领券 `auto-bind-coupons` / 积分账户 `query-my-account` / 我的券 `query-my-coupons`）；预算套餐推荐（`lib/services/recommender.dart`，按官方档位指引：≤20 小食 / 20-30 汉堡+副食 / 30-40 汉堡+小食+饮料 / 40-50 丰盛 / 50+ 丰富组合，纯 Dart 可单测）；活动日历（`campaign-calendar`，支持按日期定位，按 `structuredContent` 的 `data.dailyList[].events[]`（`articleDto`）结构化渲染）；**主题活动预约**（`query-party-city` → `query-party-store` → `query-party-store-date` → `query-party-store-session` → `party-order-create`，城市→门店→日期→场次→预约下单）；**餐品营养查询**（`list-nutrition-foods`，含能量/蛋白/脂肪/碳水/钠/钙）；**餐品详情**（`query-meal-detail` 点菜单项弹层，含**口味特调**：`modification.items[].values[]` 按 `minValues`/`maxValues` 渲染单选/多选、加价与份数，默认选中服务端 `selectedQuantity>0` 项，必选组未选时禁止加购；提交按官方格式 `items[].modification.values[]` 每项 `{code, quantity, key}`，`key` 用 `selectedKey` 原样透传，`calculate-price` 与 `create-order` 同构，实测算价接口接受；同商品不同口味在购物车分行、可「改口味」更新原行；详情无价格字段时回退菜单价）；**麦麦商城**（`mall-points-products` → `mall-product-detail`（SKU 选择、购买须知、兑换期、派对信息、限购）→ `mall-create-order`（传 `skuList[].skuId`，实物走 `addressId`；派对/活动类 `shopId≠2` 引导走主题活动预约）/ `mall-order-list` / `mall-order-detail` 订单查询）；**积分抽奖**（`query-lottery-info`（状态取 `activityStatusText`，资格按 `drawDecision.resourceEligible` 门控、展示 `nextConsumption`）/ `draw-lottery` / `query-my-prizes`）；**服务器时间**（`now-time-info` 我的页展示）；订单管理与取消（`order-list` / `query-order` / `cancel-order`）；**企业团餐（beType=6）**：门店发现复用外送逻辑，结算面板经 `query-meal-assistance` 选择助餐服务并以 `gmServiceCode` 提交 `create-order`。截至当前，官方文档所列 **33 个 MCP 工具已全部接入（100% 覆盖）**；
5. **容错与体验**：入参严格遵循**线上 `tools/list` 的 JSON Schema**（网页文档存在笔误/不一致，schema 为唯一权威；所有工具均 `additionalProperties: false`，多传字段会被服务端拒绝）——据此修正了建址城市字段名（`city`）、到店自取不传 `beCode`、优惠券只能走 `items[].couponId`、主题活动工具真名（`query-party-store-date` / `query-party-store-session`）、团餐 `gmServiceCode` 必传（来自 `query-meal-assistance`）等；官方同一套工具存在**三种响应形态**，客户端全部兼容——JSON（门店/菜单/账户/主题活动/商城/抽奖）、**Markdown 文本**（活动日历与三类优惠券，回退为 `payload['raw']` 后直接渲染）、**TOON 紧凑字符串**（营养信息，由 `lib/utils/toon.dart` 解析）；菜单为「分类引用 + `data.meals` 详情表」结构，需按 code 回查合并；**业务错误体直显**：部分工具把 `{success:false, code, message}` 作为正常结果返回（如 600057「门店可能已关闭或不在营业时间」、600048「beType=2/5/6 必传 beCode」），客户端在 `McpClient` 与 `McdService._call` 两层检测并提取 `message`+`code` 直接展示真实报错原因（不再吞成空态或泛化错误），已配回归测试；菜单价为**元字符串**（`"33.5"`、`"30"`），算价结果为**分**，分别按规则换算；工具返回字段按"常见命名依次尝试 + 原始 JSON 兜底"解析，UI 提供"原始返回数据"折叠视图；列表页进入加载一次、下拉刷新才重拉，限流自律；**错误页可返回**（`ErrorView` 增加 `onBack`，点餐页菜单加载失败时可「返回重选门店」，不再只有重试）；**大屏响应式**（`lib/utils/responsive.dart` 统一断点：compact<600 / medium<1000 / expanded<1500 / large≥1500）——**自适应 ≠ 居中限宽留白**：屏幕变大时列数与分栏随之增加，空间被真正用起来。商城网格按宽度自适应 2~8 列（内容区上限 1800）、活动卡按日期分组 1~4 列（上限 1800）、省钱页优惠券 1~3 列且宽屏拆成「积分 | 优惠券」两栏（上限 1600）、抽奖页宽屏拆成「活动/奖池 | 我的奖品」两栏、我的页宽屏拆成「会员信息 | 历史订单」两栏（上限 1400）、点餐页（上限 1600）选店门店 1~3 列、菜单按分类 1~4 列，且窗口 ≥1200 时购物车由底部条升级为**右侧常驻栏**（可改数量/删除/去结算）；窄屏（手机）一律单列堆叠，行为与旧版一致；**购物车编辑面板**（`CartSheet`：点购物车栏打开，数量步进/删除/小计/去结算，面板内说明优惠券在结算页选择）；**地址选择器**（已有地址时也可「新建收货地址」；官方无修改/删除地址接口，对话框内如实提示）；**活动卡片**（`price="0"` 不再显示误导性的「¥0起」；「立即抢购」等按钮弹窗展示官方活动页链接 `articleDto.appJumpUrl` 并支持复制——官方返回的是 app 深链，桌面端无法直接唤起；贴图改为左侧 84px 缩略图 + 右侧文字的紧凑布局）；
6. **质量保障**：`flutter analyze` 0 问题；**126 个测试全部通过**（含 41 个本地假 MCP 服务器集成测试与解析/模型/配置/组件测试，覆盖商城、抽奖、团餐助餐、活动日历结构化解析、业务错误识别、错误页返回、响应式布局、购物车编辑、口味特调与套餐换购提交、主题个性化与窄屏布局），`test/parsers_test.dart` 提供各工具返回的容错解析与 `{success:false}` 业务错误识别回归，`test/responsive_test.dart` 覆盖断点判定与列数计算，`test/cart_modification_test.dart` 覆盖口味/换购物车模型（提交格式/加价/差价/分行键/摘要），`test/theme_test.dart` 覆盖主题模式/预设/自动回退/ThemeData 构建，`test/widget_test.dart` 覆盖窄屏模式栏（360px 渲染等宽模式卡而非 SegmentedButton）、StatRow 纵排与购物车口味/搭配展示。

> 截至 2026-10-24，仓库最终状态已与上述记录一致：未引入任何第三方 MCP SDK（MCP 客户端为纯 Dart 自研；pubspec 仅 flutter / cupertino_icons / flutter_lints，以及个性化功能引入的 shared_preferences 与 dynamic_color 两个官方插件）、密钥零硬编码且 `.env` 被 git 忽略、官方文档 33 个 MCP 工具 100% 接入、业务错误体可直显、错误页可返回、真正的大屏响应式（增列 + 分栏 + 购物车右栏）、购物车可编辑、99 个测试通过。第 1–3 节记录的是 WorkBuddy 初始产出快照，用于说明本项目并非由 WorkBuddy 直接生成可用功能，而是在其骨架之上由作者完成全部实质开发。

## 4.1 增补迭代（2026-10-25）

1. **手机窄屏压缩修复**（用户截图反馈「在手机上界面就压缩成这样子了」）：
   - 点餐页顶部四段式 `SegmentedButton`（图标+文字并排）在 360px 手机上每段仅 ~85px 被压扁 → 窄屏（<600）改为**可横向滚动的等宽模式卡**（76px 宽，图标在上、文字在下，选中态描边+容器色），宽屏保留 SegmentedButton；
   - 省钱页「我的积分」三卡横排在窄屏每卡仅 ~110px → 改为**三行纵排**（新增 `StatRow` 组件：图标+标签在左、数值在右）；
   - 菜单行标签限 2 个且单个标签 110px 截断，避免超长标签撑高行；
   - 回归测试用 `tester.view.physicalSize` + `devicePixelRatio=1.0` 模拟 360/1000 宽视口（注意：新版测试框架 `tester.binding.setSurfaceSize` 对主视图不生效，需用 `tester.view`）。
2. **个性化**（`lib/app/theme.dart`）：主题模式（跟随系统/浅色/深色）+ 主题色（品牌红/麦麦黄/活力橙/抹茶绿/青碧/海蓝/魅紫/蜜桃粉 8 预设 + 「自动」动态取色）；`ThemeController extends ValueNotifier<ThemeSettings>`，SharedPreferences 持久化（失败静默降级为内存生效，便于单测）；main.dart 用 `DynamicColorBuilder` + `ValueListenableBuilder` 应用主题，全应用即时生效；「我的」页新增个性化卡片（模式分段按钮 + 圆形色板 Wrap，选中态对勾按亮度选黑白）。
   - **坑**：`dynamic_color` **2.x 依赖 material_ui，其 ColorScheme 不是 Flutter 的**（`DynamicColorBuilder` 回调签名对不上，报错信息有误导性）→ 必须用 **1.x**（1.7.0 实测正常，回调直接给 Flutter 的 ColorScheme）。
   - **坑**：`ChangeNotifier` 不是 `ValueListenable`，不能用 `ValueListenableBuilder` 直接包；要么 `ListenableBuilder`，要么让控制器继承 `ValueNotifier<快照>`（本项目采用后者）。
3. **测试**：99 → **126 个全部通过**；`flutter analyze` 0 问题。

## 4.2 再增补（2026-10-25 下午）：套餐换购（rounds）

用户截图反馈套餐详情页「还是选不了口味」——该商品的定制全在 `rounds`（换购轮次）而非 `modification`。

1. **模型**：`MenuRound{id,name,minQuantity,maxQuantity,choices}`、`MenuRoundChoice{code,name,quantity,maxQuantity,isDefault,diffPriceText,diffPriceFen,supportModify,modification}`、`CartRoundChoice{round,choice,quantity,mods}`；`CartLine` 增加 `rounds`，`lineTotalFen` 含换购差价与子项特调加价，`cartKey` 含轮次签名（不含份数），新增 `roundsText` 摘要；`toOrderItem` 提交 `roundList[{round, comboItemList:[{code,quantity,modification?}]}]`。
2. **解析**：`_parseRounds` + `_parseDiffPrice`（"+ ¥2.5"→250 / "- ¥0.5"→-50）；默认选中以 `choice.quantity>0` 为准（实测 isDefault 不可靠）。
3. **UI**：详情弹层新增「套餐搭配」区——每轮一张卡（必选/单选/最多 N 项提示），子项为可选 chip（带差价标签），已选项显示份数步进（maxQuantity=-1 不限时封顶 9），子项 `supportModify` 时渲染嵌套特调组；底部合计实时含差价；购物车/右栏/结算页展示「轮次名：子项名 ×N」摘要并支持改搭配。
4. **实测验证**（探针 + App 代码路径端到端）：鸡薯双全盒默认三轮 roundList 算价 OK（¥15.90 与用户截图一致）；龙焰芝士棒鸡腿堡三件套（多选轮次 + 主食/饮品子项特调 + 大薯条差价）算价 OK；非法 round id 报 60006（故 id 必须取自详情）。
5. **坑**：`Map['x'] = <...>[ ... ],` 列表赋值后误写逗号导致语法错误（应为分号）；轮次选中判断不能复用 CartModChoice 版的 `_isSelected`（泛型不同）。
6. **bug 走查修复**（2026-10-25 下午）：① 详情弹层单选口味组「先清组再判选中」导致可选单选组（min=0 max=1）无法取消选中——改为先记 `wasSelected` 再清组；② `_openCheckout` 复制购物车行漏传 `rounds`，用户选好的套餐搭配不会随下单提交——补上。
7. **应用图标**：工程根目录 `logo.png`（1080×1080）经 `flutter_launcher_icons`（dev 依赖，配置在 pubspec.yaml）生成 Android/iOS/Windows/macOS/Web 全平台图标；改图标后运行 `dart run flutter_launcher_icons` 即可，打包后生效。
8. **构建修复**（2026-10-25）：dynamic_color 1.7.0 硬编码 `compileSdkVersion 31` 而 androidx 传递依赖要求 ≥34 → 在根 `android/build.gradle.kts` 用 `subprojects { afterEvaluate { ... findByType(LibraryExtension)?.compileSdk = 36 } }` 统一抬升（必须注册在 `evaluationDependsOn(":app")` 之前）；同时按 8GB 物理内存的机器现实，`gradle.properties` 堆降为 `-Xmx3G`、`workers.max=2`、Kotlin 进程内编译，解决提交内存耗尽导致的 daemon 消失。`app-release.apk`（52.7MB，含新图标与全部修复）构建成功。

## 5. WorkBuddy 完整对话记录

> 参加「WorkBuddy 专项奖励」时，请从 WorkBuddy 导出本项目完整的对话上下文，粘贴至下方占位区（或作为附件与本文件一并提交）；不参加该专项奖励时可保留本占位说明。

```
【占位区：在此粘贴 WorkBuddy 导出的原始对话记录】

（粘贴位置 —— 请勿删除本代码块标记）
```
