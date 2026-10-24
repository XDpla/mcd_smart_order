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
4. **业务功能**：点餐（到店自取 `beType=1` / 得来速 `beType=5` / 麦乐送 `beType=2` 三种方式，搜店→菜单→购物车→算价（含门店券）→创建订单并返回支付链接，直达下单）；省钱（麦麦省一键领券 `auto-bind-coupons` / 积分账户 `query-my-account` / 我的券 `query-my-coupons`）；预算套餐推荐（`lib/services/recommender.dart`，按官方档位指引：≤20 小食 / 20-30 汉堡+副食 / 30-40 汉堡+小食+饮料 / 40-50 丰盛 / 50+ 丰富组合，纯 Dart 可单测）；活动日历（`campaign-calendar`，支持按日期定位）；订单管理与取消（`order-list` / `query-order` / `cancel-order`）；
5. **容错与体验**：工具返回字段按"常见命名依次尝试 + 原始 JSON 兜底"解析，UI 提供"原始返回数据"折叠视图；列表页进入加载一次、下拉刷新才重拉，限流自律；
6. **质量保障**：`flutter analyze` 0 问题；**41 个测试全部通过**（38 个单元/集成 `test(` + 3 个 `testWidgets(`），含基于本地假 MCP 服务器的客户端集成测试（`test/mcp_client_test.dart`）。

> 截至 2026-10-24，仓库最终状态已与上述记录一致：纯 Dart 实现、未引入任何第三方 MCP SDK（pubspec 仅 flutter / cupertino_icons / flutter_lints）、密钥零硬编码且 `.env` 被 git 忽略、41 个测试通过。第 1–3 节记录的是 WorkBuddy 初始产出快照，用于说明本项目并非由 WorkBuddy 直接生成可用功能，而是在其骨架之上由作者完成全部实质开发。

## 5. WorkBuddy 完整对话记录

> 参加「WorkBuddy 专项奖励」时，请从 WorkBuddy 导出本项目完整的对话上下文，粘贴至下方占位区（或作为附件与本文件一并提交）；不参加该专项奖励时可保留本占位说明。

```
【占位区：在此粘贴 WorkBuddy 导出的原始对话记录】

（粘贴位置 —— 请勿删除本代码块标记）
```
