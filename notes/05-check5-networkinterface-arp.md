# Checkpoint 5 Note（NetworkInterface / ARP）

## 目标（Goal）
在 `NetworkInterface` 中打通 IPv4 数据报与以太网帧的收发通路，并实现 ARP 解析、缓存与超时机制，使未知下一跳 MAC 时也能可靠发送。

---

## 关键不变量 / 核心概念

1. **二层发送必须有目的 MAC**
   - 上层给的是 `next_hop` 的 IP；若本地 ARP 缓存无映射，不能直接发 IPv4 帧，必须先 ARP 解析或暂存。

2. **ARP 缓存条目有生命周期**
   - `arp_table_[ip]` 仅在 `now_ms_ < arp_expire_at_ms_[ip]` 时有效；过期必须同时清理映射与到期表，避免脏状态。

3. **ARP 请求必须限速（抑制广播风暴）**
   - 同一 `ip` 的 ARP request 发送间隔 ≥ 5s（用 `arp_req_last_sent_ms_` 记录上次发送时间）。

4. **pending 队列存在的唯一目的：等待解析到 MAC**
   - `pending_dgrams_[ip]` 保存“目标 MAC 未知时”的待发送 IPv4 数据报；一旦学习到 `ip→mac` 映射，必须立刻 flush 并清理 pending 状态。

5. **解析是“协议字节流 ↔ 结构体”的双向对偶**
   - `serialize()` 把结构体写成网络字节序（大端）与分段 payload；
   - `parse()` 用 `Parser` 从分段 buffer 中按协议字段顺序读回，失败通过 `has_error()` 或异常反映。

---

## 边界条件

1. **目的 MAC 过滤**
   - 仅处理 `dst == 本机MAC` 或 `dst == 广播MAC` 的帧，否则直接丢弃。

2. **ARP request 的 target 不是本机**
   - 即使收到 ARP request，也只在 `target_ip == 本机IP` 时回复 ARP reply；否则只学习 sender 映射（可用于缓存）。

3. **ARP reply / request 都能触发学习**
   - 两者都包含 sender 的 IP/MAC；收到任一都应更新 `arp_table_` 并刷新过期时间。

4. **pending 超时丢弃**
   - 若 5s 内未完成 ARP 解析，`pending_dgrams_[ip]` 应在 `tick()` 中被清理，避免无限积压。

5. **ARP 缓存过期**
   - 30s 到期后必须删除对应映射，后续发送应重新触发 ARP。

6. **Parser 的异常路径**
   - 若 payload 中存在 borrowed `Ref<string>`，`Parser` 构造会抛异常；`recv_frame()` 必须防御，避免测试直接终止。

7. **容器遍历删除的迭代器失效**
   - `tick()` 中清理 `unordered_map` 必须使用 `it = erase(it)` 模式，不能 range-for 内 erase。

---

## 调试复盘

### Bug 1：目的 MAC 过滤条件写错导致“所有帧都被丢弃”
- **现象**：功能测试中既收不到 IPv4 数据报，也看不到 ARP 交互的后续行为；上层表现为网络完全不通。
- **定位**：检查 `recv_frame()` 的过滤逻辑，发现使用了 `dst != 本机MAC || dst != 广播MAC`。
- **结论**：逻辑应为“既不是本机也不是广播才丢弃”，必须使用 `&&`：`dst != 本机MAC && dst != 广播MAC`。

### Bug 2：tick() 的过期判断方向反了，导致 ARP 缓存“立刻过期”
- **现象**：发送端不断重复 ARP request，pending 队列无法被稳定 flush；看起来像“永远学不到 MAC”。
- **定位**：检查 `tick()` 中的过期条件，发现使用 `expire_at >= now` 时删除条目。
- **结论**：`expire_at` 是绝对到期时刻，应在 `now >= expire_at` 时删除；同时遍历删除必须用 `it = erase(it)` 避免迭代器失效。

---
