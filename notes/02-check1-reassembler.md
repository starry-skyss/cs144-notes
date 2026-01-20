# Checkpoint 1 Note（Reassembler）

## 目标（Goal）：
把乱序、重叠、重复到达的带索引子串重组为连续字节流，并在可输出时尽快写入 ByteStream，满足容量窗口约束与正确关闭语义。

## 关键不变量 / 核心概念
1. **`first_unassembled` 不变量**：始终指向“下一个应输出/写入”的全局字节索引；`[0, first_unassembled)` 的字节已按序写入到底层 ByteStream。
2. **窗口约束（available capacity）**：只接受落在 `[first_unassembled, first_unassembled + writer.available_capacity())` 的字节；窗口外字节直接丢弃。
3. **环形映射一致性**：窗口内任意全局索引 `i` 的槽位由 `offset=i-first_unassembled` 与 `idx=(head+offset)%cap` 唯一确定；`head` 始终对应 `first_unassembled` 的物理位置。
4. **去重/去重叠原则**：同一全局索引只缓存一次（`has[idx]` 作为存在性标记）；重复或重叠到达不应重复计入 pending。
5. **连续前缀推进规则**：仅当从 `head` 起连续 `has==true` 时才能写出并推进；推进后清理槽位并同步更新 `head` 与 `first_unassembled`。
6. **EOF 关闭条件**：`is_last_substring` 只记录结束位置 `end_index`；只有当 `eof_seen && first_unassembled==end_index && bytes_pending==0` 才能关闭 Writer。

## 边界条件
1. **完全过期片段**：`first_index + len <= first_unassembled`，整段应丢弃，但仍需要尝试 flush（可能已有连续前缀）。
2. **完全窗口外片段**：`first_index >= first_unassembled + cap_now`，整段丢弃。
3. **部分重叠/部分有效**：片段左侧过期或右侧超窗，需要裁剪后只缓存有效区间。
4. **重复到达**：同一索引反复到达，必须忽略已缓存位置，避免 pending 错计。
5. **重叠到达**：片段区间与缓存区间有交集，只填补缺失字节，不覆盖已知字节。
6. **洞（holes）存在**：缓存了后段但前段缺失时不能写出，等待缺口补齐后一次性推进。
7. **容量窗口变化**：`available_capacity` 随写入/读取动态变化，插入时以“当前窗口”进行裁剪与丢弃。
8. **EOF 乱序到达**：先收到 `is_last_substring` 的末尾片段也不能立刻 close，必须等全部字节拼齐并写出。

## 调试复盘
1. **Bug 1：`reassembler_win` 测试超时**
   - 现象：`check1` 只有 `reassembler_win` 超时（Timeout 15s），其余测试通过。
   - 定位：发现 `count_bytes_pending()` 采用遍历 `has_` 的 O(cap) 统计；窗口测试会高频调用该函数，cap 较大时累计耗时触发超时。
   - 结论：引入并维护 `pending_` 计数（仅在 `has` 从 false→true 与 true→false 时更新），`count_bytes_pending()` 改为 O(1) 返回，超时消失。

2. **Bug 2：过早关闭流（EOF 处理错误）**
   - 现象：最初实现中只要 `is_last_substring==true` 就 `close()`，导致乱序场景下提前关闭、后续数据无法写入。
   - 定位：EOF 语义应是“已知结束位置”，而不是“现在就结束”；需要等待缺口填完并写到末尾。
   - 结论：增加 `eof_seen/end_index`，仅当 `first_unassembled==end_index` 且 `pending==0` 时关闭 Writer，保证关闭时机正确。
