# 幂等键契约

- 决策日期：2026-08-03
- 类型：技术决策
- 状态：已决策，待实施
- 来源：qtcloud-pay `conventions.md` 幂等键约定；account/coupon/voucher/order 四个 service

## 背景

账本写入唯一入口（充值/退款/发券/结算）依赖幂等键 + `transaction.idempotency_key` 唯一约束，配合「冲突回滚视为成功」保证**不丢、不重、可查**。现状：

- 键由四个 service 手拼字符串：`"recharge:"+voucherNo`、`"refund:"+voucherNo`、`"issue:coupon:"+batchNo`、`"issue:voucher:"+batchNo`、`"settle:"+orderID`、`"settle:"+orderID+":redeem:"+kind+":"+refID`
- 命名规则（`{业务}:{业务号}` 前缀隔离键空间）只存在于 conventions.md，未代码化
- 业务号若含 `:`（如凭证号自带冒号）会破坏键空间隔离，无边界校验

## 决策

幂等键构造收敛为工具库 `pkg/idempotency` 契约：

- `Key(biz, bizNo) (string, error)`：构造 `{biz}:{bizNo}`；业务号为空或含 `:` 时返回错误（防键空间污染）
- 业务前缀常量（键空间隔离）：`recharge` / `refund` / `issue` / `settle`
- `SettleRedeemKey(orderID, kind, refID)`：结算核销复合键 `settle:{orderID}:redeem:{kind}:{ref}`
- provider 四个 service 的幂等键构造全部改用工具库；conventions.md 幂等键规则改为引用工具库文档

## 结果

- 键空间规则代码化 + 边界校验防呆，业务号含 `:` 在入口即被拒绝
- 新业务接入沿用前缀常量与构造器，不再散落字符串字面量
- 键构造与「冲突回滚视为成功」语义解耦：契约只负责键的构造与校验，幂等写入语义留在各服务
