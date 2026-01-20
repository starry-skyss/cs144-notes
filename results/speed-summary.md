# 性能基准摘要（speed）

CS144 的 speed benchmark 输出包含在 checkpoint 测试输出中（与课程默认测试流程一致），本仓库不单独重复截图。

## 证据截图（speed 输出位置）
- check0：assets/screenshots/check0_webget.png
- check1：assets/screenshots/check1_reassembler_tests.png
- check2：assets/screenshots/check2_receiver_tests.png
- check3：assets/screenshots/check3_sender_tests.png

## 关键指标摘录（throughput）

> 说明：吞吐量与机器配置、构建类型相关；以下结果来自测试输出中的 `compile with optimization` 相关 speed tests。

### ByteStream throughput
- pop length 4096：
  - 7.17 Gbit/s（check1）
  - 5.78 Gbit/s（check2）
  - 4.78 Gbit/s（check3）
- pop length 128：
  - 1.63 Gbit/s（check1）
  - 1.07 Gbit/s（check2）
  - 1.37 Gbit/s（check3）
- pop length 32：
  - 0.39 Gbit/s（check1）
  - 0.34 Gbit/s（check2）
  - 0.34 Gbit/s（check3）

### Reassembler throughput
- no overlap：
  - 16.22 Gbit/s（check1）
  - 13.40 Gbit/s（check2）
  - 17.68 Gbit/s（check3）
- 10x overlap：
  - 4.23 Gbit/s（check1）
  - 3.31 Gbit/s（check2）
  - 2.97 Gbit/s（check3）

备注：
- 上述数据为 checkpoint 测试时自动执行的 speed tests 输出摘录。
- 输出中均显示 `100% tests passed`，可用于证明实现正确性与可运行性。
