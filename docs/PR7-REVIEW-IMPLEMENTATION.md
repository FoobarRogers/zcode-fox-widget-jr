# PR #7 修复实现说明

基线：`772b4624f0bbbbffacc31e92f5f4dfe46ee04c1c`。

## 关键设计

火山域名只能确认平台，不能确认请求扣哪份额度。`source.mjs` 先识别平台，再根据 `config.json` 的 `volcBillingByProvider` 精确匹配 providerId：`plan` 返回 `volc`，其它值及未声明返回 `volc-api`。不读取配额接口的成功状态，不接受账号级 `volcPlanSubscribed` 作为计费依据。

这使同一个火山账号的按量、订阅 provider 可以共存。显示菜单只控制显示，网络错误只影响配额卡，均不改变计费属性。模型托管在火山时也不会因为名字含 GLM 而误进 GLM 套餐。

新配置需要经过 `readPluginConfig()` 的字段白名单；只改 JSON 和调用方会导致配置无效。`node lib/cli.mjs providers` 仅列出精确 ID 和解析来源，方便复制 ID，不输出发现条目的 key。

## 阅读代码的顺序

1. `credentials.mjs`：读取按 provider 的计费声明。
2. `source.mjs`：统一返回最终来源；收窄通用 coding-plan 规则和火山域名边界。
3. `turn-cost.mjs`：原有 `source === 'volc'` 预付池判断无需增加第二套声明逻辑。
4. `usage-records.mjs`：普通 Ark 与订阅都使用自己的平台标签，只有订阅清零。
5. `server.mjs` 与 `plan-balance.mjs`：会话与 GLM 套餐消费计算都提供 baseURL，保持来源一致。
6. `widget.js`：`volc-api` 显示火山本地估算；`volc` 仍显示订阅配额卡。

## 其它修复

- `balance.mjs` 的显示模式读、写两处白名单补齐 `volc`。
- AFP 窗口缺少有效 Used 时跳过，真实 Used=0 保留，数字字符串兼容保留；拒绝空串、布尔值、数组与对象转换成假零。
- 主／回退查询改成一个顺序循环：有效窗口成功返回；凭据／时钟错误立即返回；其它错误保留，避免回退空响应导致 undefined 或假报未订阅。所有最终结果都走 finish 和失败 TTL。
- 凭据读取支持注入环境与配置读取函数，使整对优先级可用假数据验证。
- 更新文档与未发布日志，不改版本号。

## 回归重点

测试不能只验证一个新的 source 字符串。selftest 用两个真实数据库 provider 验证逐轮金额与汇总：普通 Ark 金额大于零，订阅金额为零、tokens 保留；另验证 session.json 自动跟随得到相同来源。配额请求使用 https mock，含两个方向的“网络失败 + 空窗口”。

显示持久化测试分别覆盖 HTTP 写读和真实浏览器选择后重载。CLI 测试检查输出字段白名单，确保没有 API key。原有路径测试在 Linux 运行时错误要求 Windows 大小写规则，现按实际平台检查；没有修改浮层运行逻辑。

## 已知边界

- 当前是本地价目估算，不是火山账单接口；缺价目仍走原有 tokens 兜底。
- 按当前 provider 配置解释历史记录；一个 providerId 若改变计费方式，历史归类也随配置变化。建议按量与订阅使用不同 ID。
- 配额 IAM 账号与模型 provider 的账号关系由配置者确认，未新增账号绑定验证。
- 未基于未核实的端点路径自动判订阅；代理 provider 缺少平台线索时不自动升级为订阅。
- 本轮不扩展重做缓存的凭据隔离／强制刷新并发语义。

## GLM 接手

本次验证：Linux / Node v24.19.0，清空继承环境凭据、使用临时 ZCODE_HOME，并通过外部测试 preload 拒绝非 loopback socket 后，`node tools/selftest.mjs` 为 **454/454**。`git diff --check`、前端与 smoke 脚本语法检查通过。`node tools/smoke-ui.mjs` 报告 **SKIP：本机未找到 Edge/Chrome**，新增浏览器交互断言尚待 Windows 执行；不能将 SKIP 当作通过。离线 preload 是验证环境设施，没有并入产品源码。

先查看该提交的完整 diff，再在 Windows 环境运行 selftest 与 smoke-ui；重点观察火山自动跟随、菜单选择保存后重开、按量和订阅混用。确认后可把该单一提交 cherry-pick 到现有 PR 分支，无需改写先前四个提交。不要将账号级开关重新接回消费层。
