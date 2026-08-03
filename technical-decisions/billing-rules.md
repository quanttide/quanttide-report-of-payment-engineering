# billing 计费规则

- 决策日期：2026-08-03
- 类型：技术决策
- 状态：已决策，已实施
- 来源：qtcloud-pay `docs/billing.md`；`src/provider/internal/billing/`

## 背景

结算时"先抵扣什么、怎么算"是全系统最易变、最需要测试的业务规则：满减、折扣、代金券、余额的叠加顺序与边界。若把计算逻辑散落在 order 编排里，规则一变就动到事务编排代码，且无法独立测试。

## 决策

1. **纯计算模块**：`Calculate(amount int64, coupons []CouponInput, vouchers []VoucherInput, balance int64) ([]Deduction, error)` 是纯函数——无 I/O、无存储依赖，输入中立结构（不依赖 coupon/voucher 模块），输出逐项抵扣明细 `Deduction{Kind, RefID, Amount}`（`Kind`: coupon / voucher / balance）
2. **v0.1.0 默认抵扣顺序**（规则引擎后置，`BillingRule` 表预留）：
   1. 满减券：满足门槛（≤ 剩余应付）中力度最大的一张
   2. 折扣券：按 `rate` 优惠，`应付 × (100 − rate) / 100` 向下取整
   3. 代金券：逐张抵扣 `min(面值, 剩余应付)`
   4. 余额：补足剩余
3. **边界语义**：余额不足返回错误（结算拒绝，HTTP 422）；订单金额必须为正整数（分）；无任何可用券 → 全额余额
4. **规则引擎后置**：`BillingRule{Priority, Kind, Condition}` 表与 repository 先建好，v0.1.0 顺序仍由代码固定，不引入规则解析器

## 理由

- 纯函数可表驱动测试：无券/仅满减/仅折扣/满减+折扣/代金券/混合/余额不足全组合覆盖，是全系统最容易回归的部分
- 顺序与计算集中一处：order 编排只"执行计划"（按 Kind 分派核销/扣款），不"计算计划"——职责单一
- 规则引擎后置：v0.1.0 只有一套默认顺序，过早引入表达式解析是过度设计；表结构先建好保证升级路径

## 结果

- 抵扣计算可独立演进、独立测试；结算计划快照（`SettleDetail`）由此输出
- 已知遗留：默认顺序改代码固定（配置化推迟）；`Kind`/`Deduction`/`Calculate` 是跨应用共享的计费契约，候选提炼至工具库
