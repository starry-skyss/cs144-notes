# Checkpoint 3 Note（TCPSender）

## 目标（Goal）
实现一个可与对端互操作的 TCP 发送端：在窗口约束下分段发送字节流，正确处理 ACK 与重传计时器，并在异常时通过 RST 终止连接。

---

## 关键不变量 / 核心概念
1. **绝对序列号单调递增**：用 `uint64_t` 的绝对序列号（next/acked）做核心状态，实验规模下无需考虑绕回。
2. **in-flight 只由 next 与 acked 差值定义**：`sequence_numbers_in_flight = next_abs - acked_abs`，前提是始终保证 `acked_abs <= next_abs`（非法 ACK 必须忽略）。
3. **窗口约束针对“序列号空间”**：窗口消耗包含 `payload` 字节数 + `SYN`(1) + `FIN`(1)；`RST` 不占序列号空间。
4. **outstanding 队列队头=最早未确认段**：只重传队头段；新发送段入队，重传不重复入队。
5. **ACK 推进即“正向进展”**：只要 ACK 变大，就重置计时器、RTO 与连续重传次数，不能依赖“是否 pop 了整段”来判断。
6. **RST/error 终止语义**：进入 error 或收到对端 RST 后，后续发送应携带 RST，并停止正常发送/重传逻辑（避免继续消耗 ByteStream 数据）。

---

## 边界条件
1. **接收端通告窗口为 0**：发送端视作 1（persist-like 行为），允许重传队头，但**不**进行指数退避与连续重传计数递增。
2. **SYN/FIN 的序列号占用**：SYN 仅发送一次且占 1；FIN 仅在 ByteStream finished 且窗口仍有余量时发送并占 1。
3. **可发送数据为 0**：若既无 payload 又不能发 SYN/FIN，则不应发送空段（避免死循环/无意义段）。
4. **最大负载限制**：每段 payload 不超过 `TCPConfig::MAX_PAYLOAD_SIZE`，同时不能超过剩余窗口 `remain`。
5. **ACK 缺失（ackno 不存在）**：只更新窗口，不更新确认相关状态，不弹队列、不重置计时器。
6. **非法 ACK（ack > next_abs）**：必须忽略，防止 `in_flight` 下溢导致窗口计算失真。
7. **部分确认（ACK 落在段内部）**：队列按“整段被确认”(`segment_end <= ack`) 才弹出；但若 ACK 推进仍需重置计时器/RTO。
8. **RST 到来或本端 error**：发送端不再从 ByteStream 取数据；可发一个 RST 空段通知对端并停止计时器。

---

## 调试复盘
### Bug 1：ACK 推进后 RTO/计时器不重置
- **现象**：即使对端持续 ACK，新段仍频繁触发超时重传，`consecutive_retransmissions` 异常增长，RTO 失控翻倍。
- **定位**：在 `receive()` 中先 `ackno_abs_ = max(ackno_abs_, ack)`，再判断 `if (ack > ackno_abs_)`，导致“ACK 推进”条件永远为假；同时曾误写 `rto_ms_ = 0`。
- **结论**：必须先计算 `ack_advanced = (ack > ackno_abs_)` 再更新 `ackno_abs_`；ACK 推进时将 `rto_ms_` 重置为 `initial_RTO_ms_` 并清零 `timer_ms_` 与 `con_retransmissions_`。

### Bug 2：RST 状态下仍消耗 ByteStream 数据导致“吞数据”
- **现象**：进入 error/RST 后，应用侧写入的数据被 `pop` 掉但未被正确纳入发送序列/未入 outstanding，表现为数据丢失、状态机卡住或测试异常。
- **定位**：在 `push()` 中先 `peek/pop` 构造段，再在发送前才设置 RST 并 `break`，导致数据已从 ByteStream 消耗但发送路径提前终止。
- **结论**：RST/error 必须在 `push()` 开头短路处理：直接发送一个 RST 空段并返回，禁止继续 `peek/pop` 或推进序列号与队列。

---
