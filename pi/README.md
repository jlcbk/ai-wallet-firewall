# Raspberry Pi：联网网关边界

Raspberry Pi 是联网和复杂编排层。它可以访问 RPC、获取链上状态、构造完整 unsigned transaction、向 ESP32 提交请求，并保存决定元数据和广播记录。

它不是签名权威，也不是硬策略的所有者。

## 可以负责的工作

- 与区块链 RPC 和索引服务通信；
- 获取 nonce、gas、费用和网络状态；
- 根据用户或 AI/Agent 的意图构造完整交易；
- 将完整交易交给 ESP32，而不是提交预先计算的 digest；
- 接收 ESP32 的签名结果并广播；
- 保留可审计的请求、决定和广播状态。

## 明确限制

- 把 AI/Agent 指令、RPC 返回值和网络数据都当作不可信输入；
- 不在源码、日志或配置中保存私钥、助记词、provisioning secret、admin/root key 或 RPC secret；
- 永远不要求 ESP32 签任意 hash；
- 不把主机侧交易摘要当作最终事实；
- 不暴露运行模式下修改 ESP32 硬策略的操作；
- 不因为网络重试、超时或用户界面状态而把一次拒绝自动改成允许。

Pi 侧实现必须兼容 protocol/ 中的 Schema，但最终的交易解析和硬策略决定始终属于 ESP32。
