# transaction 交易账本

- 决策日期：2026-08-03
- 类型：技术决策
- 状态：已决策，已实施
- 来源：qtcloud-pay `docs/transaction.md`；`src/provider/internal/transaction/`

## 背景

所有账本变更（充值/退款/消费/发券/核销）需要一个唯一写入入口与统一记录形状，否则各模块各自建表写账，"不丢、不重、可查"无法作为系统性保证落地，对账与客诉也没有单一事实来源。

## 决策

1. **不可变账本（append-only）**：`Transaction` 只插入、不更新、不删除——Service 只暴露 `Append`/查询方法，无 Update/Delete 面；记录含 `AccountID / Type / Amount / BalanceAfter / OrderID / IdempotencyKey / Note / CreatedAt`
2. **账本写入唯一入口**：所有写账本路径（充值/发券/消费/核销）都必须经 `transaction.Append`，配合 `idempotency_key` 唯一索引
3. **幂等语义 = 冲突回滚视为成功**：事务内先查后插，唯一约束冲突时**整体回滚并返回成功**（`ErrDuplicateKey → nil`），调用方不重试——并发下后到者回滚自己的全部修改，让先到者生效
4. **余额快照**：`BalanceAfter` 由调用方在同事务内、拿到账户行锁后计算（锁后余额可信），供对账与客诉
5. **交易类型契约化**：类型常量（recharge/refund/consume/issue/redeem）与余额语义（`AffectsBalance`/`SignedAmount`——发券/核销是信息性记录，不参与余额求和）已提炼至工具库 `pkg/ledger`，provider 内保留类型别名

## 理由

- append-only：交易一旦落库不可篡改，审计、对账（余额 = Σ 带符号交易）、客诉回溯都以账本为唯一依据
- 幂等键 + 唯一约束在数据库层兜底"不重"：即使业务侧先查后插存在竞态窗口，唯一索引仍保证最终只插入一条
- 类型契约进工具库：账务、对账等后续应用读取同一交易语义，不在各应用重复实现

## 结果

- 余额一致性校验、对公核对、账单导出全部基于账本推导，无需信任余额字段
- 已知遗留：`Transaction` 结构契约（JSON 形状）仍锁在 provider（gorm 模型），跨应用读账本时需提炼至工具库
