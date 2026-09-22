# ai-wallet-firewall：面向 AI/Agent 的受控加密货币签名防火墙

`ai-wallet-firewall` 是一个面向 AI Agent、自动化脚本和个人链上工具的**受控加密货币签名防火墙**设计项目。

它要解决的不是“如何让 AI 帮忙发一笔交易”，而是一个更具体、更危险的问题：

> 当 AI、Raspberry Pi 或联网服务被误导、被提示注入，甚至已经被攻破时，如何仍然不让它们绕过本地硬规则，拿着私钥签出一笔未经授权的交易？

本项目采用分层、隔离的架构：让 AI 参与理解意图和准备交易，让 Raspberry Pi 负责联网和复杂编排，但把最终的交易解析、硬策略判断和签名边界放到离线 ESP32 上；真正的私钥则保存在可拆卸的 SE050 安全元件中，永不导出。

## 一句话理解

```text
AI / Agent
    -> Raspberry Pi
       联网、访问 RPC、构造完整的 unsigned transaction
    -> ESP32-S3-RLCD
       离线解析完整交易、执行硬策略、计算 signing hash
    -> 可拆卸 SE050
       保存不可导出的私钥并执行受控签名
```

Raspberry Pi 可以提出“请签这笔完整交易”，但不能提出“请签这个我算好的 hash”；它也不能在运行模式下修改 ESP32 的硬策略。ESP32 必须自己解析完整的 unsigned transaction，自己计算签名哈希，并在交易未知、格式异常、解析有歧义或超出策略时直接拒绝。

## 为什么需要这个项目

传统的热钱包脚本、自动化交易机器人或“AI + RPC + 钱包私钥”方案，通常把太多权力放在同一台联网主机上：

- AI 可能受到网页、合约数据、聊天内容或提示注入影响，产生错误的交易意图；
- Raspberry Pi 或其上的服务一旦被攻破，攻击者可能直接请求任意签名；
- 如果主机只把一个预先计算好的 digest 交给签名器，签名器无法知道 digest 对应的真实收款地址、金额和 calldata；
- 如果策略文件也由联网主机管理，攻击者可以先把“每日限额”和“允许合约”改掉，再发起看似正常的交易；
- 对未知交易类型、未知合约方法或解析不完整的交易采取“尽量继续”，会把不确定性变成签名权限。

因此，本项目把**联网便利性**和**最终签名权**拆开：上游可以复杂、灵活、可自动化；下游必须简单、受限、可拒绝。

## 核心架构

| 组件 | 所处位置 | 可以做什么 | 明确不能做什么 |
| --- | --- | --- | --- |
| AI / Agent | 最上游 | 理解用户意图、准备候选交易、解释结果 | 不能接触私钥，不能决定最终签名，不能绕过设备策略 |
| Raspberry Pi | 联网网关 | 访问 RPC、获取 nonce 和费用、构造完整交易、提交请求、记录审计信息 | 不能修改 ESP32 硬策略，不能要求签任意 hash，不能成为签名权威 |
| ESP32-S3-RLCD | 离线安全边界 | 解析完整 unsigned transaction、显示关键字段、执行硬策略、计算签名 hash | 生产固件不启用网络，运行模式不接受策略修改 |
| 可拆卸 SE050 | 硬件信任锚 | 保存私钥并执行受控签名 | 不能导出私钥，不能替代 ESP32 做交易语义判断 |
| 智能合约钱包 | 链上资产账户 | 保存资产、验证 signer、执行阈值和恢复规则 | 不能依赖单一不可恢复的 SE050 私钥 |

这个结构的关键不是“使用了多少设备”，而是**每一层只拥有完成自己工作所需的权限**。即使 Raspberry Pi 失陷，也不应因此获得修改硬策略或任意签名的能力。

## 设备安全与资产恢复是两层防护

本项目同时解决两个不同的问题：

### 设备层：SE050 被拆走怎么办

SE050 不是一个可以随便插到其他主机上的 USB 签名棒。生产配置应让它与指定 ESP32 进行 host binding，使用每台设备独有的 SCP03 密钥，并强制所有会话加密和认证；签名对象还要使用最小权限策略。ESP32 侧则需要 Secure Boot、Flash Encryption、关闭调试入口、模块身份检查以及拔插后的重新认证。

这样做的目标是：**单独拿走 SE050，不能直接把它变成可用签名器**。这些措施不能替代链上撤销，所以失窃时仍然必须走资产账户的 signer rotation。

### 资产层：SE050 损坏或丢失怎么办

重要资产不直接放在由单个 SE050 私钥控制的不可恢复 EOA 中，而放入具备 signer 替换和恢复能力的智能合约钱包：

```text
日常交易：ESP32 + SE050 signer
资产账户：Smart Contract Wallet
恢复路径：独立离线 recovery signer
```

SE050 只负责日常签名；恢复 signer 不在 Pi、ESP32 或同一个模块中。SE050 丢失后，可以用恢复 signer 在链上移除旧 signer、添加新设备，而不需要导出旧私钥。完整设计见 [资产恢复模型](docs/recovery-model.md)。

## 一笔交易如何流动

1. 用户或 AI/Agent 表达交易意图。
2. Raspberry Pi 访问链上 RPC，获取必要的 nonce、费用和网络信息。
3. Raspberry Pi 构造一份完整的 unsigned transaction，而不是只发送摘要。
4. Pi 通过本地传输把交易请求交给 ESP32。
5. ESP32 检查帧长度、字段、版本、chain ID、交易类型和边界。
6. ESP32 自己解析全部交易字段和 calldata，并重建用于签名的规范表示。
7. ESP32 用本地硬策略检查目标地址、方法选择器、金额、gas、每日限额、nonce 和特殊操作。
8. 如果交易未知、畸形、不支持、超限或有歧义，ESP32 直接拒绝，不接触 SE050。
9. 只有通过检查的交易，才由 ESP32 自己计算 signing hash，并请求 SE050 签名。
10. ESP32 返回签名和必要的决定元数据，Pi 将签名提交给智能合约钱包并负责广播和记录。

整个流程中，**没有一个上游组件能够把任意 digest 直接塞给 SE050**。

## 两种工作模式

### 运行模式（Run Mode）

这是日常交易模式。ESP32 只开放完成交易审核所需的最小接口，例如提交完整交易、读取状态和获取决定结果。以下操作不属于运行模式：

- 安装、重置或放宽硬策略；
- 请求任意 digest 签名；
- 通过网络更新生产固件；
- 在策略更新过程中继续正常签名。

### 管理模式（Admin Mode）

策略修改必须由明确的物理动作触发，例如断电后按住实体按键再上电。进入管理模式后：

1. 屏幕明确显示管理状态；
2. 普通签名功能暂停；
3. ESP32 校验策略包、版本、签名和回滚规则；
4. 设备记录新策略的 hash；
5. 完成后退出管理模式，重新进入运行模式。

具体的按键、显示和管理员密钥仪式仍属于待实现的硬件与固件工作，但“运行模式不能改策略”是不可削弱的安全边界。

## 这个项目带来的优势

### 1. 把 AI 的灵活性限制在签名边界之外

AI 可以帮助解释自然语言意图、准备交易和生成策略草案，但它不应直接拥有密钥，也不应拥有最后的签名权限。这样可以把“AI 可能犯错”与“AI 可以造成不可逆资产损失”拆成两个不同的问题。

### 2. Raspberry Pi 被攻破后仍有第二道边界

Pi 负责联网，所以它本来就比离线设备更容易受到攻击。架构不假设 Pi 永远安全，而是假设它可能失陷，并要求 ESP32 仍然独立解析交易、执行硬策略。

### 3. 防止“签名摘要”和“用户理解内容”脱节

ESP32 不接受主机直接提供的 hash。它看到的是完整交易，并自己计算签名内容。未来如果支持更多链或更多交易类型，每种类型都必须有自己的有界解析器、规范编码和负面测试向量。

### 4. 策略修改拥有物理边界

硬策略不是放在 Pi 上的一份普通配置文件。运行时 Pi 不能修改它；需要修改时，必须进入一个明确、可观察、暂停签名的管理流程。

### 5. 可拆卸安全元件便于维护与分层验证

SE050 与主控分离，可以独立验证私钥不可导出、模块缺失时拒绝签名、替换和恢复流程，而不把所有信任都压在联网主机或 ESP32 软件上。

### 6. 让安全要求变成可审查的仓库资产

项目不只保存代码，还保存威胁模型、信任边界、协议 Schema、策略模型和验证记录。后续无论由人、Codex、Claude Code 还是其他 Agent 继续，都可以从同一份安全基线开始，而不是依赖聊天记录中的隐含约定。

这些优势是架构目标，不等于本仓库已经完成安全审计。物理提取、供应链、固件替换、侧信道、SE050 配置和每条链的解析正确性仍需要独立实现与验证。

## 仓库结构

```text
ai-wallet-firewall/
├── README.md                           # 项目介绍、架构和使用边界
├── AGENTS.md                           # 人和 Agent 必须遵守的安全不变量
├── SECURITY.md                         # 秘密管理与漏洞报告规则
├── docs/
│   ├── architecture.md                 # 组件、信任边界和数据流
│   ├── threat-model.md                 # 资产、攻击者假设和安全目标
│   ├── signing-flow.md                 # 逐步签名流程与拒绝条件
│   ├── policy-model.md                 # 硬策略、评估顺序和管理模式
│   └── recovery-model.md                # 智能合约钱包与 signer 恢复流程
├── protocol/
│   ├── README.md                       # 协议基线说明
│   ├── policy.schema.json               # ESP32 硬策略数据模型
│   └── transaction-request.schema.json  # 完整 unsigned transaction 模型
├── pi/README.md                        # Raspberry Pi 联网网关边界
├── esp32/README.md                     # ESP32-S3-RLCD 离线安全边界
└── hardware/se050-module/README.md     # 可拆卸 SE050 模块边界
```

协议字段暂时保留稳定、可编程的英文命名，例如 `chain_id`、`transaction_type` 和 `max_value_wei_per_tx`；面向人阅读的解释、决策和安全约束以中文为主。

## 当前阶段与明确非目标

当前仓库是**设计与协议基线**，不是可以直接承载真实资产的生产钱包，也不是经过审计的硬件签名设备。当前版本暂不声称已经完成：

- ESP32 生产固件；
- SE050 真实 provisioning 和密钥仪式；
- 完整的 EVM 或其他链交易解析器；
- 实物 PCB、电气、掉电、拆卸和恢复验证；
- 物理攻击、侧信道和供应链安全验证。

本项目也不会把以下内容放进仓库：真实私钥、助记词、SE050 provisioning secret、admin/root key、RPC secret 或真实设备备份。详见 [SECURITY.md](SECURITY.md)。

## 从哪里开始

- 想理解整体设计：阅读 [架构说明](docs/architecture.md) 和 [威胁模型](docs/threat-model.md)。
- 想理解一次交易如何被拒绝或签名：阅读 [签名流程](docs/signing-flow.md)。
- 想讨论策略字段和管理边界：阅读 [策略模型](docs/policy-model.md) 与 [协议基线](protocol/README.md)。
- 想理解 SE050 丢失后的资产恢复：阅读 [资产恢复模型](docs/recovery-model.md)。
- 想让 Agent 继续开发：先阅读 [AGENTS.md](AGENTS.md)，再检查实现是否保留所有安全不变量。

## 搜索关键词

中文搜索：`AI 钱包`、`AI 签名防火墙`、`加密货币签名安全`、`树莓派 钱包`、`ESP32 离线签名`、`SE050 私钥不可导出`、`SE050 模块被盗`、`智能合约钱包恢复`、`signer rotation`、`硬策略 钱包`、`交易解析 签名`、`防止任意 sign(hash)`、`提示注入 链上交易`。

English search: `AI wallet firewall`, `agent-controlled crypto signing`, `Raspberry Pi wallet gateway`, `ESP32 offline transaction parser`, `SE050 non-exportable private key`, `transaction-aware signer`, `fail-closed crypto wallet`, `no arbitrary sign hash`.

## 相关技术参考

- [NXP AN12662：Binding a host device to EdgeLock SE05x](https://www.nxp.com/docs/en/application-note/AN12662.pdf)：SE050 与指定主控绑定、SCP03 和安全启动思路。
- [NXP AN12413：SE050 APDU Specification](https://www.nxp.com/docs/en/application-note/AN12413.pdf)：安全对象、认证、对象策略和密钥生命周期。
- [Safe Smart Account 概览](https://docs.safe.global/advanced/smart-account-overview)：多 signer、threshold、owner 替换和智能账户模型参考。
- [Ethereum Account Abstraction](https://ethereum.org/roadmap/account-abstraction)：智能合约账户与密钥恢复能力的背景说明。

## 许可证与安全提醒

本仓库目前聚焦设计和安全边界。任何真实资产部署前，都应完成代码审查、协议测试、固件验证、硬件验证和独立安全评估；通过 Schema 校验、编译或模拟器测试，都不能单独证明交易语义、实时行为或物理设备安全。
