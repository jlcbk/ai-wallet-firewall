# 协议基线

本目录保存 Raspberry Pi 与 ESP32 之间共享的第一版控制面 Schema。

- transaction-request.schema.json：描述一份完整的 unsigned transaction 请求，不是 raw digest 请求。
- policy.schema.json：描述 ESP32 硬策略的数据模型，暂不等同于最终的认证 provisioning 包。

## 重要边界

Schema 只是字段和格式校验工具，不是授权证明。ESP32 固件仍必须自己执行：

- 有界的帧和交易解析；
- 支持的交易类型检查；
- 规范编码与 signing hash 计算；
- 硬策略比较；
- 未知输入 fail closed；
- 与 SE050 之间的受控签名调用。

当前没有规定最终的线缆编码和传输帧格式。后续实现可以选择合适的本地传输，但不得因此引入任意 sign(hash) 路径，也不得让 Pi 的摘要取代 ESP32 的完整解析。

## 字段命名

协议字段暂时使用稳定的英文命名，便于固件、脚本和多语言工具共同使用；面向人的说明以中文为主。新增字段时必须同步更新 Schema、协议文档、正面样例和负面测试向量。
