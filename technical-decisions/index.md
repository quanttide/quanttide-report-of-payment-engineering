# 技术决策

记录量潮支付工程领域的技术决策报告。

## 内容

- 技术选型与架构决策
- 协议与接口设计决策
- 数据模型与存储决策
- 安全与合规决策
- 性能与可靠性决策

## 决策记录

### provider 账本核心

- [provider-architecture.md](provider-architecture.md) — 账本核心架构：单向依赖、四层结构、账本写入唯一入口、无后台任务、渠道刻意平行
- [account-wallet.md](account-wallet.md) — 账户与余额：余额是交易投影、同事务更新、凭证号幂等
- [transaction-ledger.md](transaction-ledger.md) — 交易账本：append-only、写入唯一入口、冲突回滚视为成功
- [coupon.md](coupon.md) — 优惠券：批次幂等、状态机与惰性过期、核销
- [voucher.md](voucher.md) — 代金券：面值抵现、与 coupon 同构、不做部分使用
- [billing-rules.md](billing-rules.md) — 计费规则：纯计算模块、默认抵扣顺序、规则引擎后置
- [order-settlement.md](order-settlement.md) — 订单与结算：单事务编排、账户行锁串行化、结算快照
- [reconciliation.md](reconciliation.md) — 对账与可查：一致性校验、对公核对、账单导出
- [channel-layer.md](channel-layer.md) — 支付渠道层：Provider 接口抽象、透传 client + 适配器翻译、v0.2.0 回调入账

### 跨应用契约

- [idempotency-key-contract.md](idempotency-key-contract.md) — 幂等键契约（前缀隔离键空间、边界校验防污染）
- （支付状态契约已归档至工具库 `quanttide-pay-toolkit/packages/go/docs/dev-guide/status.md`）
- （金额决策已归档至工具库 `quanttide-pay-toolkit/packages/go/docs/dev-guide/money.md`）

## 命名约定

决策报告文件用英文命名（小写连字符），不带头日期，如 `money-value-object.md`。
