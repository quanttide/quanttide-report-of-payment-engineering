# 技术决策

记录量潮支付工程领域的技术决策报告。

## 内容

- 技术选型与架构决策
- 协议与接口设计决策
- 数据模型与存储决策
- 安全与合规决策
- 性能与可靠性决策

## 决策记录

- [idempotency-key-contract.md](idempotency-key-contract.md) — 幂等键契约（前缀隔离键空间、边界校验防污染）
- （支付状态契约已归档至工具库 `quanttide-pay-toolkit/packages/go/docs/dev-guide/status.md`）
- （金额决策已归档至工具库 `quanttide-pay-toolkit/packages/go/docs/dev-guide/money.md`）

## 命名约定

决策报告文件用英文命名（小写连字符），不带头日期，如 `money-value-object.md`。
