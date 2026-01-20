# 总览（CS144 Networking Lab）

本文档用于说明 Stanford CS144 Networking Lab 的整体模块结构、数据流与关键概念（公开笔记，不含实验解答代码/实现片段）。

---

## 1. 完成情况

已完成所有可独立完成的 checkpoint：

- ✅ check0：webget（最小 HTTP 客户端）+ ByteStream（容量约束）
- ✅ check1：Reassembler（乱序/重叠重组）
- ✅ check2：TCP Receiver（ACK/窗口/unwrap + RST/error 传播）
- ✅ check3：TCP Sender（窗口约束、重传计时器、RST 终止）
- ✅ check5：NetworkInterface / ARP（二层封装、ARP 缓存/超时/抑制）
- ✅ check6：Router（多接口 IPv4 转发：LPM + TTL）

说明：
- check4：无需要实现的代码
- check7：需要合作完成（未包含）

---

## 2. 协议栈结构与模块职责

CS144 的实验可以视作从应用层到二层的一个简化互联网协议栈实现，核心组件包括：

- **ByteStream（check0）**  
  有容量约束的字节缓冲区，提供 push/peek/pop 等基本语义；其“背压”能力保证任意时刻缓冲不超过 capacity。

- **Reassembler（check1）**  
  将乱序、重复、重叠到达的“带索引数据片段”重组为连续字节流；只输出可连续拼接的前缀，并严格受窗口/容量约束。

- **TCP Receiver（check2）**  
  处理入站 TCP 段，解决 32 位序列号回绕（wrap/unwrap），把 TCP 序列空间对齐为字节流 index；生成 ACK 与窗口通告，并处理 RST 与 ByteStream error 的双向传播。

- **TCP Sender（check3）**  
  在发送窗口约束下分段发送字节流，维护 outstanding 队列与重传计时器（RTO），在 ACK 推进时完成计时器与退避状态重置；异常场景通过 RST 终止连接。

- **NetworkInterface / ARP（check5）**  
  打通 IPv4 数据报与以太网帧的收发通路；实现 ARP 解析、缓存、过期与请求限速；在目的 MAC 未知时对数据报进行 pending 暂存并在学习到映射后 flush。

- **Router（check6）**  
  多接口 IPv4 路由器：对入站 IPv4 数据报执行最长前缀匹配（Longest-Prefix Match，LPM）选路，处理 TTL 递减与到期丢弃，并从指定接口转发至下一跳或直连目的地址。

---

## 3. 核心概念与“不变量”视角

实验中多个模块的正确性都可用“不变量”来描述与检查：

- **容量/窗口不变量（ByteStream / Reassembler / Receiver）**
  - ByteStream：bytes_buffered() 始终不超过 capacity，push 受 available_capacity 限制
  - Reassembler/Receiver：只接收窗口范围内数据，窗口外直接丢弃，避免内存失控与性能退化

- **连续前缀不变量（Reassembler）**
  - 只要存在 gap 就不能输出；仅当从 first_unassembled 起形成连续前缀时才能写入并推进

- **序列号映射不变量（TCP Receiver）**
  - Wrap32 unwrap 必须返回“与 checkpoint 距离最近”的绝对序列号，保证回绕情况下 ACK 映射稳定
  - SYN/FIN 各占用 1 个序列号，stream index 对齐必须一致

- **发送窗口与 in-flight 不变量（TCP Sender）**
  - in-flight 由 next_abs 与 acked_abs 的差值定义；非法 ACK（ack > next_abs）必须忽略
  - 窗口消耗包含 payload + SYN(1) + FIN(1)；RST 不占用序列号空间
  - outstanding 队列队头为最早未确认段：只重传队头，重传不重复入队

- **协议错误传播不变量（Receiver/Sender）**
  - RST 与 ByteStream error 必须双向传播：收到 RST 立刻置流 error；本端流 error 也应对外体现 RST 语义

- **二层必须有目的 MAC（NetworkInterface/ARP）**
  - 上层只给 next_hop IP；若无 ARP 映射必须先解析或暂存，不能“直接发 IPv4 帧”

- **路由选路不变量（Router）**
  - 所有匹配路由中选择 prefix_length 最大的一条（LPM）
  - TTL<=1 必须丢弃；可转发包必须先 TTL-- 再发送
  - 无匹配必须丢弃（不能默认乱发）

---

## 4. 数据流

### 4.1 端到端收发路径（简化）
1. 应用产生数据 → 写入 ByteStream
2. TCP Sender 在窗口约束下生成 TCP 段（含重传逻辑）
3. TCP 段封装为 IPv4 数据报 → 交给 NetworkInterface
4. NetworkInterface 通过 ARP 获取目的 MAC → 封装以太网帧发送
5. 对端接收帧 → 解封装 IPv4 → 交给 TCP Receiver
6. TCP Receiver 将数据交给 Reassembler → 按连续前缀写入 ByteStream → 应用读取

### 4.2 转发路径（Router）
1. 接口收到以太网帧 → 解封装为 IPv4 数据报
2. Router 执行 LPM 查表选择出口接口/下一跳
3. TTL--，若到期则丢弃
4. 交给出口 NetworkInterface：触发 ARP（如未命中）并发送

---

## 5. 工程化验收与调试方法

- 验收方式：以官方测试套件为标准（check0/1/2/3/5/6 均通过）
- 调试方法：
  - sanitizer（ASan/UBSan）定位越界/未定义行为
  - tcpdump/Wireshark 抓包验证 ARP/转发等协议现象
  - 最小失败用例复现 + 日志记录关键状态量（窗口/计时器/缓存/TTL 等）

---

## 6. 文档索引

- 各 checkpoint 详细笔记：见 `notes/01~06-*.md`
- 调试手册：`notes/90-debugging-playbook.md`
- 经验总结：`notes/99-lessons-learned.md`
