# provider 账本核心架构

- 决策日期：2026-08-03
- 类型：技术决策
- 状态：已决策，已实施
- 来源：qtcloud-pay `docs/index.md`、`docs/conventions.md`；`src/provider/`

## 背景

v0.1.0 在既有 `channel` 渠道模块旁新增账本核心（account/transaction/coupon/voucher/billing/order/reconciliation 六模块），由 `cmd/server` 统一组装。需要一组横切约束决定模块边界、依赖方向与协作方式，否则各模块容易长出各自的存储、事务与错误处理风格。

## 决策

1. **账本核心 = account + transaction**：`transaction` 是最底层模块（被所有写账本模块依赖），`account` 是客户资金入口；其余模块依赖它们，依赖图单向无环
2. **模块统一四层结构**：`transport / service / repository / model + gorm/`；repository 接口方法以 `*gorm.DB` 为首参（`*gorm.Tx` 内嵌 `*gorm.DB`，事务与共享连接可直接互换），跨模块写在单事务内完成、事务由编排方开启
3. **存储双引擎**：repository 只写一套 GORM 实现，开发 SQLite / 生产 PostgreSQL 由 `cmd/server` 按 `DB_DRIVER`/`DATABASE_URL` 启动时选择，AutoMigrate 注册全部模型（版本化迁移后置）
4. **账本写入唯一入口**：所有账本变更（充值/发券/消费/核销）必须经 `transaction` 模块写入，配合幂等键与唯一约束——不丢、不重、可查
5. **同事务更新**：余额、券状态与交易在同一数据库事务内更新——不错
6. **金额全链路整数分**（详见 money 决策）；`channel` 渠道层 float64 元是已知遗留（F2），新代码一律以分传输
7. **无后台任务**：券过期用惰性流转（读取时更新状态），对账按需调用，无调度器
8. **渠道刻意平行（v0.1.0 MVP 边界）**：`channel` 不依赖账本模块、支付成功不入账，`-channel` flag 默认为空（生产 FC 部署即纯账本 API）；业务闭环（支付→入账）推迟 v0.2.0
9. **配置全走环境变量、零配置文件、零启动校验**：对齐 FC 环境变量注入；渠道缺配置在 `NewProvider` 时即报错，不带病启动

## 理由

- 账本写入唯一入口 + append-only（见 transaction 决策）使"余额=交易投影"可被对账验证，审计与客诉有单一事实来源
- 四层结构 + repository 以 `*gorm.DB` 为首参：单元测试可 mock repository，跨模块编排（结算）可在单事务内组合任意模块，无需分布式事务
- 双引擎单实现：开发与生产同构，方言差异收敛在启动装配
- 零配置 + 无后台任务：对齐 FC 无状态部署模型，最小化运维面

## 结果

- 新增功能 = 新增模块：按四层模板建目录、写 `docs/<module>.md`、按依赖图接线
- 共享领域契约逐步提炼至工具库（已迁：money / ledger / status；候选：transaction 账本契约、billing 计费计算、券契约）
- 已知遗留：渠道层 float64 金额（F2）、无支付回调路由（F3）、券过期惰性流转的"读时写"代价
