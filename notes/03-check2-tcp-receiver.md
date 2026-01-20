# Checkpoint 2 Note （tcp_receiver)

## 目标（Goal）
实现 TCP 接收端的核心逻辑：正确处理 32 位序列号回绕、乱序/重叠段重组，生成 ACK/窗口通告，并完成 RST 与 ByteStream error 的双向传播，且在压力测试下不过时。

---

## 关键不变量 / 核心概念
1. **序列号映射不变量**：`Wrap32::unwrap(ISN, checkpoint)` 必须返回“与 checkpoint 距离最近”的绝对序列号候选（以 `2^32` 为周期），保证回绕情况下映射稳定。
2. **stream index 对齐**：TCP 序列空间中 SYN/FIN 各占用 1 个序列号；payload 对应的 `stream index` 必须与 “SYN 后第一个字节为 index 0” 一致，ACK 计算需同步考虑 SYN/FIN。
3. **接收窗口语义**：Reassembler 只接收落在窗口 `[first_unassembled, first_unassembled + available_capacity)` 的数据；窗口外数据直接丢弃（避免破坏容量约束与性能退化）。
4. **重组推进条件**：只有当 `head_`（即 `first_unassembled` 对应位置）已被填充，才可能推进并向 ByteStream 写入连续前缀；否则只缓存，不推进。
5. **pending 的一致性**：`pending_` 必须始终等于“已缓存但尚未写入 ByteStream 的字节数”，作为 O(1) 的权威统计，避免重复扫描统计导致性能问题。
6. **错误传播一致性**：ByteStream `error` 与 TCPReceiver `RST` 必须双向对应：流出错则对外 RST；收到 RST 则置流 error。

---

## 边界条件
1. **SYN 之前到来的段**：在 ISN 未建立前到来的数据段不可用于重组与 ACK（通常忽略，直到收到 SYN）。
2. **32 位回绕场景**：序列号跨 `2^32` 回绕时，`unwrap` 需选择最接近 checkpoint 的候选，避免跳到错误周期导致 ACK 异常。
3. **乱序与重叠段**：同一位置多次到达、部分覆盖、完全重复都必须正确去重；重复段不能导致 pending 计数错误或性能崩溃。
4. **窗口外段（左侧/右侧）**：完全在左侧（已组装）或完全在右侧（超出窗口）应直接丢弃；部分重叠需要裁剪后插入。
5. **FIN 的处理**：FIN 表示 EOF 位置，不等同于“最后一个段一定带全部数据”；必须在所有数据到齐且 pending 为 0 后再 close。
6. **容量为 0 或接近满**：available_capacity 很小时仍要保证不越界写 ring buffer，且不做无意义的扫描/拼接。
7. **RST/stream error**：RST 到来应立即将 ByteStream 置 error；ByteStream set_error 后 `send()` 需返回 `RST=true`。

---

## 调试复盘
### Bug 1：`recv_reorder_more` 超时（压力测试 15s watchdog）
- **现象**：`recv_reorder_more` 长时间运行后 Timeout，功能测试大多通过但压力场景失败。
- **定位**：热路径在 Reassembler 的标记数组与索引计算：
  - `std::vector<bool>` 的 bit-proxy 访问在高频读写中开销巨大；
  - 大量 `% cap_` 取模出现在循环里，sanitized/bug-checkers 下被放大；
- **结论**：将 `has_` 改为 `std::vector<uint8_t>`，在热路径去除取模（用条件减法/递增指针）后，压力测试通过且吞吐显著提升。

### Bug 2：`recv_special`（RST 与 error 状态传播不完整）
- **现象**：
  - `Stream error -> RST flag`：ByteStream `set_error` 后，期望 `TCPReceiverMessage.RST=true`，实际为 false；
  - `RST flag set -> stream error`：收到带 RST 的段后，期望 ByteStream `has_error=true`，实际为 false。
- **定位**：
  - `send()` 未将底层 error 映射到 `RST`；
  - `receive()` 未处理入站 `RST`（需要将 ByteStream 置 error）；同时 `Reassembler` 对外只暴露 `const writer()`，不能从外部调用 writer 的 `set_error()`，应改用可变 `reader()` 设置 error。
- **结论**：在 `send()` 中设置 `rec.RST = stream.has_error()`；在 `receive()` 起始处遇到 `message.RST` 时调用 `reassembler_.reader().set_error()` 并返回，完成双向传播后测试通过。

---
