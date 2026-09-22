# 资产恢复模型

## 核心决定

本项目不把“SE050 中的一把私钥”当作资产唯一控制权。SE050 的运行密钥可以保持不可导出，同时把资产放进具备 owner / signer 替换能力的智能合约钱包。

```text
日常签名：ESP32 + SE050
资产账户：Smart Contract Wallet
恢复控制：独立离线恢复签名者
```

这样，SE050 损坏、丢失或被盗时，恢复签名者可以在链上移除旧签名者、加入新的签名设备，而不需要从旧 SE050 导出私钥。

## 角色分离

| 角色 | 位置 | 作用 | 不应拥有的东西 |
| --- | --- | --- | --- |
| 日常签名者 | ESP32 + SE050 | 执行经过硬策略的日常交易 | 不拥有恢复密钥，不负责修改恢复成员 |
| Raspberry Pi | 联网网关 | 构造交易、提交签名、广播和记录 | 不成为智能账户 owner，不持有恢复密钥 |
| 恢复签名者 | 独立离线硬件或密封备份 | 丢失设备后更换 signer、暂停或迁移资产 | 不与 Pi、ESP32 或 SE050 共用密钥 |
| 智能合约钱包 | 链上账户 | 保存资产并执行 owner、threshold、恢复规则 | 不能依赖单一不可恢复的 SE050 私钥 |

策略管理员、SE050 运行绑定密钥和资产恢复密钥是三个不同的信任角色，不能因为“都是管理员”而合并。

## 推荐账户形态

对于高价值资产，优先采用多签或带恢复机制的智能合约钱包：

```text
Smart Account
├── 日常 signer：SE050 signer
├── 恢复 signer：离线硬件钱包
└── 可选 signer：第二个独立恢复位置
```

高价值账户建议使用 2-of-3 或同等的阈值模型。日常 signer 被盗时，攻击者不能单独替换 owner 或转走全部资产；恢复 signer 可以移除旧 signer 并添加新 signer。

如果自动化要求日常 signer 单独执行低风险交易，则应在智能账户层使用有界的 session key、allowance 或 spending limit，而不是让 SE050 成为可以无限制执行任意合约调用的唯一 owner。任何这类模块都必须单独审计，因为模块本身可能成为新的资产控制边界。

## 正常交易流程

1. AI/Agent 向 Pi 表达意图。
2. Pi 构造一笔面向智能合约钱包的完整交易。
3. ESP32 解析完整交易并执行硬策略。
4. SE050 使用不可导出的运行密钥签名。
5. Pi 将签名提交给智能合约钱包或其 relayer。
6. 智能合约钱包验证 signer、threshold、nonce 和账户规则后执行交易。

SE050 签的是智能账户交易请求，不是资产恢复密钥，也不是任意由 Pi 提供的摘要。

## SE050 损坏或丢失后的恢复流程

```text
发现模块损坏 / 丢失 / 被盗
        ↓
停止 Pi 的日常签名和广播
        ↓
将旧 signer 标记为 revoked（链下与设备侧）
        ↓
恢复 signer 发起 owner / signer rotation
        ↓
加入新的 ESP32 + SE050 signer
        ↓
重新设置 threshold、限额和恢复规则
        ↓
用小额交易验证新设备
        ↓
恢复日常运行
```

恢复操作不需要读取旧 SE050 的私钥。它依赖的是智能合约钱包原本就存在的恢复控制路径，因此必须在第一次存入重要资产之前完成部署和演练。

## 被盗时的处置

如果只是 SE050 被盗：

- 立即停止 Pi 继续向旧 signer 提交交易；
- 不尝试通过 Pi “覆盖”或导出旧私钥；
- 使用恢复 signer 尽快移除旧 signer；
- 检查链上 pending transaction、allowance 和 owner 变更；
- 给新的硬件 signer 重新做 SE050 host binding；
- 重新验证 smart account 的 threshold 和恢复成员。

如果 SE050 与 ESP32 主板一起丢失，则按整台签名设备失窃处理。Secure Boot、Flash Encryption 和 SE050 host binding 可以提高攻击成本，但不能替代链上 signer rotation。

## 密钥生命周期

### 日常运行密钥

优先在 SE050 内部生成，设置不可导出和最小对象权限，只通过绑定的 ESP32 使用。该密钥不提供传统的助记词备份。

### 恢复密钥

在独立、离线且可验证的硬件或密封介质中生成和保存。恢复密钥不能存储在 Pi、ESP32、同一个 SE050 模块、Git 仓库或普通云盘中。

### 新设备密钥

新 SE050 使用新的设备身份、新的 host binding 和新的 signer 公钥。不要为了“方便替换”而复制旧 SE050 的运行绑定密钥到多台设备。

## 不能解决的问题

- 智能合约钱包的 bug 仍可能导致资产损失；
- 未审计的 recovery module 可能成为新的攻击面；
- 如果恢复 signer 也丢失，仍可能无法恢复；
- 某些链不支持同样的智能账户能力；
- 账户部署、owner rotation 和恢复交易需要 gas 与可用的广播路径；
- 如果 SE050 signer 被设计成无限权限 owner，多签和恢复机制可能仍无法阻止日常盗签。

因此，智能合约恢复是资产恢复层，不是对 ESP32 交易解析和 SE050 绑定的替代品。

## 最低验收条件

在投入重要资产前，至少要证明：

1. 没有旧 SE050 私钥时，恢复 signer 可以移除旧 signer；
2. 失窃的旧 signer 不能单独改变 owner 或 threshold；
3. 新 SE050 可以完成新的 host binding 和身份登记；
4. Pi 没有恢复密钥，也不能直接调用智能账户 owner 管理接口；
5. 恢复流程在测试网和小额资产上完整演练过；
6. 恢复路径不会引入任意 sign(hash) 接口。

## 参考

- [Safe Smart Account 概览](https://docs.safe.global/advanced/smart-account-overview)
- [Safe owner 移除接口](https://docs.safe.global/reference-smart-account/owners/removeOwner)
- [Ethereum Account Abstraction](https://ethereum.org/roadmap/account-abstraction)
