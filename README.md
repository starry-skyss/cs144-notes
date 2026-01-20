# Stanford CS144 Networking Lab — 公开笔记（不含解答 / No Solutions）

本仓库仅包含 Stanford CS144 Networking Lab 的**公开、非解答型笔记**。  
为遵守课程要求并维护课程作为教学工具的价值：  
**本仓库不包含任何实验解答代码、关键实现片段、patch/diff、或可直接复现解答的内容**。

- 课程主页：https://cs144.stanford.edu  
- 实现代码仓库：**私有**

## 完成情况

已完成所有可独立完成的 checkpoint：

- ✅ check0：webget
- ✅ check1：Reassembler
- ✅ check2：TCP Receiver
- ✅ check3：TCP Sender
- ✅ check5：NetworkInterface / ARP
- ✅ check6：Router（IPv4 转发：最长前缀匹配 LPM）

## 测试命令

以下为官方构建/测试目标：

```bash
cmake -S . -B build
cmake --build build

# 运行指定 checkpoint 的测试
cmake --build build --target check[checkpoint_num]

# 运行全部测试
cmake --build build --target test

# 运行速度基准测试
cmake --build build --target speed
