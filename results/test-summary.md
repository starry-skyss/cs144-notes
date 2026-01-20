# 测试结果摘要（CS144，公开笔记不含解答）

本文件仅记录 Stanford CS144 Networking Lab 的测试验收结果与证据截图，不包含任何实验解答代码/实现片段。

---

## 1. 完成情况（可独立完成部分）

已完成以下 checkpoint，并通过官方测试目标：

- check0：webget
- check1：Reassembler
- check2：TCP Receiver
- check3：TCP Sender
- check5：NetworkInterface / ARP
- check6：Router（IPv4 forwarding：Longest Prefix Match + TTL）

说明：
- check4：无需要实现的代码
- check7：需要合作完成（未包含）

---

## 2. 证据截图（PASS 输出）

- check0：PASS  
  证据：assets/screenshots/check0_webget.png

- check1：PASS  
  证据：assets/screenshots/check1_reassembler_tests.png

- check2：PASS  
  证据：assets/screenshots/check2_receiver_tests.png

- check3：PASS  
  证据：assets/screenshots/check3_sender_tests.png

- check5：PASS  
  证据：assets/screenshots/check5_arp_tests.png

- check6：PASS  
  证据：assets/screenshots/check6_router_tests.png

---

## 3. 运行命令

构建：

    cmake -S . -B build
    cmake --build build

运行指定 checkpoint 测试：

    cmake --build build --target check[checkpoint_num]

---

## 4. 备注

- 性能基准测试（speed）的输出包含在部分 checkpoint 测试输出中（如 check2/check3 截图），详见 results/speed-summary.md。
