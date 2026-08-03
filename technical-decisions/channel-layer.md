# channel 支付渠道层

- 决策日期：2026-08-03
- 类型：技术决策
- 状态：已决策，已实施（v0.2.0 回调入账待实施）
- 来源：qtcloud-pay `docs/channel.md`；`src/provider/internal/channel/`；渠道状态统一契约见工具库 `dev-guide/status.md`

## 背景

需要接入外部支付渠道（微信 JSAPI 公众号/小程序、支付宝网页支付 PC）。渠道报文、签名、状态码各不相同且受外部演进约束，若直接散落在业务代码里，渠道切换与渠道升级都会牵动账本逻辑。

## 决策

1. **可替换渠道层 + Provider 接口抽象**：`Provider{Name, Pay, Query, Refund}` 是渠道中立契约，微信/支付宝各一个适配器实现（编译期断言 `var _ Provider`）；HTTP 层（`transport.go`）只面向接口
2. **双层结构：透传 client + 适配器翻译**：`channel/wechat`、`channel/alipay` 底层 client 透传渠道原生报文（`TradeState`/`trade_status`）；协议翻译（渠道码 → 统一状态）收敛在 `adapters.go` 适配器边界——统一状态契约（`PaymentStatus`/`RefundStatus` 小写枚举 + `Parse*` 解析函数 + 未知码显式报错）见工具库 status 决策
3. **金额单位与状态码的边界翻译**：微信分 ↔ 元、支付宝字符串元在适配器内完成；**已知遗留 F2**：渠道层仍用 float64 元且 `int(x*100)` 截断存在精度缺陷（与 money 整数分决策冲突，待改 int64 分）
4. **配置环境变量注入**：渠道密钥全走环境变量，缺配置在 `NewProvider` 时即报错，不带病启动
5. **v0.1.0 刻意平行（MVP 边界）**：渠道不写 `order` 表、支付成功不入账、`-channel` flag 默认为空——生产 FC 部署即纯账本 API；"结算即支付"仅限余额/券抵扣，外部支付闭环推迟 v0.2.0
6. **v0.2.0 接入路径（已规划）**：新增支付回调 handler——验签（`ParseNotify`/`VerifyNotify`）→ `transaction.Append`（type=recharge，幂等键 = 渠道交易号）→ 自动入账，替代手动登记；**模型不变，变的只是交易来源**

## 理由

- 双层结构把"渠道差异"锁在适配器：底层 client 保持原生报文（可直接对接渠道文档），统一语义只出现一次（工具库状态契约），未知渠道码显式报错而非静默降级
- 刻意平行：账本核心跑通前不扩展渠道能力（架构决策 8），v0.1.0 交付纯账本 API，支付闭环是 v0.2.0 的独立增量

## 结果

- 渠道可替换：新增渠道 = 新 client 包 + 新适配器，账本与 HTTP 层零改动
- 已知遗留（ROADMAP F2–F7）：float64 金额截断精度、无回调路由（F3）、退款参数/退款状态/错误处理/证书序列号（F4–F7）——回调入账实施时一并治理
