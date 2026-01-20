# 调试手册（Debugging Playbook）

本文档总结我在 Stanford CS144 Networking Lab 实验过程中常用的定位与验证方法（公开笔记，不含实验解答代码/实现片段）。

---

## 1. 调试目标

CS144 的大部分失败并不是“协议大方向错”，而是由以下问题导致：

- 边界条件不完整（乱序、重叠、窗口、回绕、TTL 等）
- 状态机不一致（ACK 推进、窗口推进、FIN/SYN 计数）
- 定时器/重传行为不符合规范
- 资源约束/容量控制不正确
- 二层 ARP 解析与缓存时序问题导致包“发不出去/发不对”

因此调试重点是：**复现 → 观测 → 对照不变量 → 最小修复 → 全量回归**。

---

## 2. 常用工具与使用方式

### 2.1 官方测试套件
- 运行单个 checkpoint：
  - `cmake --build build --target check[checkpoint_num]`

### 2.2 抓包（tcpdump / Wireshark）
适用场景：

- ARP：request/reply 是否出现、是否重复发送、映射是否更新
- Router：是否真的发生转发、TTL 是否递减、不同接口的输出是否符合预期
- webget：HTTP 请求是否发送/是否收到响应

抓包的意义：验证“协议现象”，不依赖实现细节。

### 2.4 最小化日志（Minimal Logging）
日志是调试状态机类问题的关键，但需要克制：

- 只打印关键状态量（如 index/ack/window/timer/cache hit）
- 避免输出大量 payload
- 优先在状态转移点打印（例如 ACK 推进、队列出入、TTL drop）

---

## 3. 常见问题分类与排查重点

### 3.1 Reassembler：乱序 / 重叠合并类
常见现象：
- 输出缺字节或重复字节
- gap 存在时仍输出（不应发生）
- EOF 处理异常（提前关闭或永不关闭）

排查重点：
- “只输出连续前缀”的不变量是否被破坏
- overlap 合并后的区间是否正确（避免漏合并或误合并）
- capacity/窗口之外的数据是否被正确丢弃

---

### 3.2 TCP Receiver：ACK/窗口与 wrap/unwrap 类
常见现象：
- ACK 不推进 / ACK 推进过头
- window 通告异常（负值/不合理变动）
- SYN/FIN 导致的 off-by-one

排查重点：
- wrap/unwrap 是否稳定选择“距离 checkpoint 最近”的绝对序号
- SYN/FIN 的序列号消耗是否一致（各占用 1）
- ACK 生成是否严格基于“已组装连续前缀”的位置

---

### 3.3 TCP Sender：发送窗口 / outstanding / 重传定时器类
常见现象：
- 误重传：没超时却重传、频繁重传
- 漏重传：超时后不重传
- 对 ACK 处理错误（非法 ACK 未忽略 / 队列未弹出）
- window=0 等边界场景表现异常

排查重点：
- outstanding 队列：队头是否始终是“最早未确认段”
- ACK 推进时 outstanding 是否正确出队
- 定时器启动/重置条件是否正确：
  - 发送第一个 outstanding 时启动
  - ACK 推进且仍有 outstanding 时重置
  - outstanding 为空时停止
- RTO 退避与重置规则是否符合预期

---

### 3.4 NetworkInterface / ARP：二层解析与缓存时序类
常见现象：
- “包发不出去”（没有目的 MAC）
- ARP request 疯狂刷屏（抑制策略缺失）
- cache 过期后短暂不可达或等待队列不 flush

排查重点：
- ARP cache 命中/过期是否正确
- 未命中时是否暂存 pending 数据报并发送 ARP request
- request 发送是否有抑制（同一目标不应毫秒级重复广播）
- 收到 ARP reply 后是否刷新缓存并 flush pending

---

### 3.5 Router：LPM / TTL / 多接口转发类
常见现象：
- 选错路由（最长前缀匹配错误）
- TTL 未递减或 TTL 到期仍转发
- 多接口路径混乱（转错接口/下一跳错误）

排查重点：
- LPM：必须选择 prefix_length 最大匹配项
- TTL：
  - 转发前 TTL 必须递减
  - TTL<=1 必须丢弃
- next_hop：
  - 有 next_hop → 发往 next_hop
  - 无 next_hop → 直连发往目的地址
- 与 ARP 的衔接是否合理（出口接口决定二层解析目标）

---

## 4. 通用调试流程

1. **复现**：锁定最小失败测试用例
2. **确认预期行为**：对照 checkpoint 规范/协议规则，明确不变量
3. **观测关键状态**：最小日志输出关键状态量
4. **定位根因**：判断属于哪一类问题（边界/状态机/计时器/缓存）
5. **最小修复**：只修影响不变量的最小点，避免引入新 bug
6. **回归验证**：该 checkpoint 全量测试

---

## 5. 我在 CS144 中形成的验证习惯

- 任何修复都必须能用测试复现并验证消失
- 对“状态机类 bug”一定要用日志记录关键状态量，而不是盲改
- 对“链路层/转发类 bug”优先抓包确认“协议现象”是否正确

---

## 6. 补充的个人复盘

