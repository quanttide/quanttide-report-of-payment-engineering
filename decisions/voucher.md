# voucher 代金券

- 决策日期：2026-08-03
- 类型：技术决策
- 状态：已决策，已实施
- 来源：qtcloud-pay `docs/voucher.md`；`src/provider/internal/voucher/`

## 背景

M2 需要直接抵现的营销手段。代金券**本身就是钱**（固定面值），结算时直接抵减应付款项——与优惠券（抵扣规则）语义不同，但生命周期管理（发放/过期/核销）与 coupon 高度同构。

## 决策

1. **固定面值（分）**：`Voucher{ID, AccountID, BatchNo, Amount, Scope, ProductID, ExpiresAt, Status, UsedAt, OrderID}`——与 `Coupon` 同构，去掉 `rate`/`threshold`
2. **抵扣额 = `min(面值, 剩余应付)`**：由结算计划（billing）计算，逐张抵现
3. **发放/幂等/状态机/惰性过期与 coupon 同构**：批次幂等由发券交易幂等键 `issue:voucher:{batch_no}` 保证；`issued → used` / `issued → expired` 惰性流转
4. **v0.1.0 不做部分使用**：抵现后未用完的面值不退还（力度设计，规则引擎后置）

## 理由

- 与 coupon 同构：发放、幂等、过期、核销的代码路径与心智模型完全对称，降低维护成本；差异仅在"面值直减"与"规则抵扣"
- 不做部分使用：避免"券上余额"这一新状态机（部分使用 → 部分退还 → 过期），v0.1.0 力度设计刻意简化

## 结果

- 抵现/过期/核销可追溯（发券交易 + `UsedAt`/`OrderID` 关联）
- 已知遗留：`Scope`/`Status` 枚举与 coupon 各复制一份（漂移风险）；部分使用不支持（规则引擎后置时重估）
