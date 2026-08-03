# 技术决策

记录量潮支付工程领域的技术决策报告。

## 内容

- 技术选型与架构决策
- 协议与接口设计决策
- 数据模型与存储决策
- 安全与合规决策
- 性能与可靠性决策

## 决策记录

- [money-integer-cents.md](money-integer-cents.md) — 内部金额 int64（分）存储、API 元传输，`pkg/money.Cents` 零误差转换

## 命名约定

决策报告文件用英文命名（小写连字符），不带头日期，如 `money-integer-cents.md`。
