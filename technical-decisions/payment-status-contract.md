# 支付状态统一契约与渠道映射

- 决策日期：2026-08-03
- 类型：技术决策
- 状态：已决策，待实施
- 来源：qtcloud-pay `internal/channel/adapters.go`；工具库 `quanttide-pay-toolkit/packages/go/pkg/status`

## 背景

渠道层支付/退款状态表示不统一：

- 微信支付：透传原生 `TradeState`（`SUCCESS`/`REFUND`/`NOTPAY`/`USERPAYING`/`CLOSED`/`PAYERROR`…）
- 支付宝：本地 map 映射为大写枚举（`PENDING`/`SUCCESS`/`CLOSED`/`UNKNOWN`），未知码兜底 `UNKNOWN`
- 两套码并存、语义未固化；`UNKNOWN` 兜底会掩盖新状态出现的问题，错误语义可能静默入账

## 决策

1. **统一状态契约**：支付单、退款单状态以工具库 `pkg/status` 枚举为准（小写英文、可直接存库、与渠道报文流转一致）：
   - `PaymentStatus`：created / paying / succeeded / failed / closed / refunding / refunded
   - `RefundStatus`：created / processing / succeeded / failed
2. **渠道码严格映射**：`pkg/status` 新增 `ParseWechatTradeState` / `ParseAlipayTradeStatus`——渠道原始码 → 统一状态
3. **未知码显式报错**：映射失败返回错误，不用 `UNKNOWN` 兜底——与金额边界拒绝未知币种同一原则：新渠道状态出现时显式暴露，而非静默降级

## 实现约定

微信 `TradeState` → `PaymentStatus`：

| 渠道码 | 统一状态 |
|--------|----------|
| `SUCCESS` | succeeded |
| `REFUND` | refunding |
| `NOTPAY` | created |
| `USERPAYING` | paying |
| `CLOSED` | closed |
| `PAYERROR` | failed |
| 其他 | 错误 |

支付宝 `trade_status` → `PaymentStatus`：

| 渠道码 | 统一状态 |
|--------|----------|
| `TRADE_SUCCESS` | succeeded |
| `TRADE_FINISHED` | succeeded |
| `WAIT_BUYER_PAY` | created |
| `TRADE_CLOSED` | closed |
| 其他 | 错误 |

- `IsValidPaymentStatus` / `IsValidRefundStatus` 供存库前防御校验
- provider `channel/adapters.go` 删除本地映射表，改用工具库映射

## 结果

- 渠道码与存储值分离：存库/流转统一用小写值，渠道差异收敛在适配器边界
- 新渠道状态显式报错，不静默入账错误语义
- 支付网关、账务等后续应用直接复用同一状态契约
