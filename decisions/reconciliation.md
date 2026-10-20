# reconciliation 对账与可查

- 决策日期：2026-08-03
- 类型：技术决策
- 状态：已决策，已实施
- 来源：qtcloud-pay `docs/reconciliation.md`；`src/provider/internal/reconciliation/`

## 背景

M4 是账本可靠性的最后一环：账本系统"不丢、不重、不错"不能只靠写入路径自证，需要独立验证手段与可查接口——余额与交易是否一致（一致性校验）、对公打款是否全部入账（对账核对）、客户账单（可查）。

## 决策

1. **一致性校验（不错）**：逐账户比对「余额字段」与「Σ(充值) − Σ(余额支付)」（由交易账本推导），输出差异清单 `Discrepancy{AccountID, Balance, Expected}`——余额是交易的投影，这条决策让"余额错"成为可检测事件
2. **对公打款核对**：上传银行流水 CSV（`voucher_no,amount_cents,date`，金额为分）→ `parseBankCSV` 纯函数解析（严格校验：整数分、必填字段）→ 与充值交易按凭证号匹配（幂等键 `recharge:{voucher_no}`）→ 输出 `BankReport{Matched, Unmatched+原因}`
3. **账单导出（可查）**：`Statement` = 期初余额 + 流水（带运行余额 `RunningBalance`，只对影响余额的交易累加）+ 期末余额，复用 transaction 查询与 `AffectsBalance`/`SignedAmount` 语义
4. **按需调用，无调度器**：三接口均为显式调用（`GET /reconcile/consistency`、`POST /reconcile/bank`、`GET /accounts/{id}/statement`），不做定时任务（见架构决策 7）

## 理由

- 校验基于账本推导而非对称实现：不信任余额字段、不复制一套求和逻辑——直接用 `pkg/ledger` 的带符号金额语义，验证的是"写入路径是否破坏了投影不变式"
- 对公核对利用幂等键做匹配键：凭证号既是入账幂等键又是对账匹配键，一套键两用，无需额外关联字段
- 无调度器：对账频率与业务节奏绑定（人工/外部触发），避免引入定时任务运维面

## 结果

- "不错"可验证：一致性校验能把投影破坏暴露为差异清单；对公打款可核对到每张凭证
- 已知遗留：`Discrepancy`/`BankReport`/`Statement` 是对账契约，账务/结算应用读取账单时候选提炼至工具库；无自动调度（对账周期自动化后置）
