# Checkpoint 0 Note (Webget)

## 目标（Goal）
在 CS144 提供的 TCP 抽象之上完成一个最小可用的 HTTP 客户端 `webget`，并实现具有容量约束的 `ByteStream` 读写缓冲区，使相关单元测试通过。

---

## 关键不变量 / 核心概念
1. **ByteStream 容量约束（背压）**：任意时刻 `bytes_buffered() <= capacity_`，写入只能发生在 `available_capacity()` 允许范围内。
2. **累计计数的单调性**：`bytes_pushed()` / `bytes_popped()` 为累积值，只增不减；与当前缓冲区大小无关。
3. **peek/pop 的零拷贝语义**：`peek()` 只“观察”不“消费”，`pop(len)` 才移除前缀数据；消费量应按实际可用数据裁剪。
4. **流结束的判定**：`close()` 表示写端终止；`Reader::is_finished()` 当且仅当“已 close 且缓冲区为空”。
5. **HTTP 报文格式严谨性**：请求行与头部必须遵循 CRLF（`\r\n`）行结束，且头部结束必须有空行（`\r\n\r\n`）。
6. **网络 I/O 的终止条件**：读取响应应持续进行直到对端关闭连接（EOF）；写入请求应保证完整发送（避免部分写入）。

---

## 边界条件
1. **push 超容量**：当 `data.size() > available_capacity()` 时，仅写入前 `available_capacity()` 字节，其余本次丢弃。
2. **push 在容量为 0/缓冲满**：`available_capacity()==0` 时写入 0 字节（no-op），不应报错。
3. **close 后 push**：close 后不再写入（no-op），保持 closed 状态，不把它当作 error。
4. **pop 超过缓冲区长度**：实际弹出量为 `min(len, bytes_buffered())`，避免越界。
5. **空缓冲 peek**：返回空 `string_view`（或等价表现），不得触发异常。
6. **is_finished 的区别**：缓冲区为空但未 close ⇒ “暂时无数据”；缓冲区为空且已 close ⇒ “彻底结束”。
7. **HTTP 请求行空格/头部格式**：`GET <path> HTTP/1.1` 两个空格位置必须正确；`Connection: close` 冒号后空格建议保留。
8. **外网依赖的测试不稳定性**：`t_webget` 依赖访问 `cs144.keithw.org:80`，网络出口变化可能导致 RST，从而使总目标 `check0` 失败但本地逻辑正确。

---

## 调试复盘
### Bug 1：webget 连接被重置（RST），导致 `t_webget` 失败
- **现象**：`webget` / `curl` 显示 `Connection reset by peer`，`check_webget` / `check0` 中 `t_webget` 报 output mismatch。
- **定位**：抓包显示三次握手成功、HTTP GET 已发送并被 ACK，随后对端 IP 立即发 RST（非本机/NAT 注入）。
- **结论**：不是实现错误，而是网络出口/对端策略导致的拒绝；更换可用出口后恢复，测试通过。

### Bug 2：`pop()` 内部使用错误类型导致潜在溢出/不一致
- **现象**：在实现 `pop(len)` 时曾用 `int` 保存实际弹出长度，存在截断/溢出风险，且与 `std::string::size_type` 不匹配。
- **定位**：检查接口类型：`len` 为 `uint64_t`，`buffer_.length()` 为 `size_t`，`erase()` 参数为 `size_type`；混用 `int` 会造成隐式转换问题。
- **结论**：将实际弹出长度统一为无符号/size 类型并用 `min()` 裁剪，确保删除与计数更新一致。
