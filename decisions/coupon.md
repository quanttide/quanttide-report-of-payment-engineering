# coupon 优惠券

- 决策日期：2026-08-03
- 类型：技术决策
- 状态：已决策，已实施
- 来源：qtcloud-pay `docs/coupon.md`；`src/provider/internal/coupon/`

## 背景

M2 需要营销抵扣手段。优惠券**本身不代表钱**，是一条抵扣规则（按比例优惠或满足门槛后减额）——与代金券（本身就是钱）在语义和结算行为上不同，需要独立的生命周期管理：发放（批量、幂等）、过期流转、结算时核销。

## 决策

1. **两种类型**：折扣券（`rate` 整数百分比，9 折 = 90 = 省 10%，`应付 × (100 − rate) / 100` 向下取整）/ 满减券（应付 ≥ `threshold` 时减 `amount`）；适用域 `Scope`（all/cloud/course/data/product）+ `ProductID` 匹配
2. **批量发放、批次幂等**：一次发放 `count` 张 + 一个 `batchNo`，共写入一条发券交易（type=issue）；幂等由发券交易幂等键 `issue:coupon:{batch_no}`（全局唯一）保证——`batch_no` 本身只建普通索引（一批多张券共用批次号，不能唯一约束），`CountByBatch` 作防御性检查
3. **状态机**：`issued → used`（结算核销）/ `issued → expired`（超过有效期）；**过期惰性流转**——不做定时任务，读取与结算时校验 `ExpiresAt`，发现过期时更新状态
4. **核销（供结算调用）**：`Use` 校验 `status=issued` 且未过期，置 `used`、关联订单；`GetForUpdate` 双保险防并发重复核销（主依赖是账户行锁串行化，见 order 决策）
5. **抵扣额计算归 billing**：券只提供自身属性（rate/threshold/amount），抵扣顺序与金额计算由 `billing.Calculate` 决定

## 理由

- 批次幂等以账本为准：发券交易幂等键是唯一权威，`batch_no` 索引只作防呆，避免"一批多张"与唯一约束的冲突
- 惰性过期：无后台任务约束（见架构决策 7），过期流转的代价是"读时写"，换取零调度器运维
- 核销不自行算钱：抵扣规则集中在 billing 纯函数，券模块只负责状态与属性，职责单一

## 结果

- 发放/核销/过期全部可追溯（发券交易 + `UsedAt`/`OrderID` 关联）
- 已知遗留：`Type`/`Scope`/`Status` 枚举与 voucher 模块各复制一份（同一词汇两处定义，存在漂移风险）；部分使用（未用完不退还）不支持
