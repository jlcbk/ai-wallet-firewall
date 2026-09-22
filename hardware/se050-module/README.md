# 可拆卸 SE050 模块

SE050 是本项目的硬件信任锚。它保存不可导出的私钥，只在 ESP32 已经解析并授权完整交易之后执行受控签名。

## 模块边界

- 模块从 ESP32 组件中可拆卸，便于受控 provisioning、维护、替换和单独验证；
- Raspberry Pi 和 AI/Agent 永远不能得到私钥；
- ESP32 不能向 Pi 暴露 raw-hash 签名能力；
- provisioning secret、admin/root key 和真实设备凭据永远留在 Git 之外；
- 模块缺失、未初始化、认证失败或状态异常时，系统必须拒绝签名。

## 防止模块被单独拿走后继续签名

生产设备不能把 SE050 当作任何主机都能调用的通用签名棒。应至少完成以下配置：

1. SE050 与指定 ESP32 建立 host binding；
2. 使用每台设备独有的 Platform SCP03 密钥，并在出厂前替换默认密钥；
3. 强制 Platform SCP，拒绝未加密、未认证的 APDU；
4. 为签名私钥对象设置认证要求，禁止读取、导出、覆盖和删除；
5. 将绑定密钥放在 ESP32 的受保护存储中，不放在 Raspberry Pi；
6. ESP32 侧启用 Secure Boot、Flash Encryption，并关闭不需要的调试和下载入口；
7. 启动和重新插入时检查 SE050 身份、绑定会话和 provisioning 状态。

NXP 的绑定方案说明了使用唯一 SCP 密钥、强制 SCP 会话以及把 SE050 与特定 MCU 配对的流程：[AN12662 — Binding a host device to EdgeLock SE05x](https://www.nxp.com/docs/en/application-note/AN12662.pdf)。

这层防护的目标是让“只拿到 SE050”不能直接得到签名服务；它不能替代智能合约钱包的链上 signer rotation。

## 为什么采用可拆卸设计

可拆卸并不意味着“方便任何人拿走私钥”，而是把安全元件、主控和网关分成可独立验证的边界。后续可以分别测试：SE050 的密钥不可导出、模块存在检测、掉电行为、替换流程和恢复流程，而不必把全部假设藏在 Raspberry Pi 软件中。

如果 SE050 与 ESP32 主板一起被盗，应按整台签名设备失窃处理：停止日常签名，使用独立 recovery signer 在智能合约钱包中移除旧 signer，再为新模块建立新的 host binding。可拆卸性解决维护问题，不能单独解决资产恢复问题。

## 待实现工作

- 确定连接器、电气接口和模块存在检测；
- 定义 SE050 object ID、访问条件和 provisioning 仪式；
- 验证复位、拔出、替换、未初始化和恢复行为；
- 证明缺少或异常的模块会 fail closed；
- 建立不包含真实秘密的模拟和负面测试向量；
- 演练 SE050 丢失后的链上 signer rotation，而不是尝试导出旧私钥；
- 验证只拿到模块时，普通主机、Pi 和未绑定 ESP32 都无法调用签名对象。
