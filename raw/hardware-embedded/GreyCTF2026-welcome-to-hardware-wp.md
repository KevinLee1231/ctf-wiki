# Welcome to Hardware

## 题目简述

这是 GreyFlag ESP32-C6 徽章的入门题。题目提供完整合并固件 `greyflag_merged.bin`，要求先把固件写入徽章，再通过 USB 串口进入挑战菜单。第一项是五道关于板卡器件与接口的选择题，全部答对后显示 flag。

## 解题过程

按 `use.txt` 使用 `esptool` 从地址 `0x0` 写入合并镜像：

```bash
esptool.py -p <serial-port> write_flash 0x0 greyflag_merged.bin
```

重启后用 ESP-IDF 串口监视器连接：

```bash
python -m esp_idf_monitor <serial-port>
```

在主菜单进入 `Challenges`，再选择 `Welcome to GreyFlag`。固件中的题目与正确答案依次是：

| 问题 | 正确答案 | 依据 |
| --- | --- | --- |
| 徽章使用什么微控制器？ | ESP32-C6 | 芯片丝印及题目参数 |
| 便携供电来源是什么？ | LiPo battery | 背面锂电接口/丝印 |
| 数字音频接口是什么？ | I2S | 固件音频驱动使用 I2S |
| 徽章上有多少颗 LED？ | 5 | 正面可见器件数量 |
| ESP32-C6 原生支持哪种 802.15.4 Mesh 协议？ | Thread | C6 的 802.15.4/OpenThread 功能 |

全部回答正确后串口输出：

```text
grey{my_baby_steps}
```

题目把完整固件交给了选手，因此也能离线验证答案。对镜像做 ASCII 字符串扫描，可以同时找到题干、四个选项、正确答案说明和 flag；完整 flag 位于镜像字节偏移 `82276`：

```bash
strings -a greyflag_merged.bin | grep -E 'What microcontroller|grey\{'
```

这不是猜测或从 README 抄答案，而是对实际刷入镜像内容的直接核验。

## 方法总结

本题的目标是熟悉一条基础嵌入式工作流：识别串口、刷写合并固件、监控 USB 输出并通过菜单交互。因为固件未隐藏字符串，静态扫描还是一个可靠的旁证，也说明发布固件中不应以明文保存真正需要保密的数据。实体解法和离线固件核验得到相同结果，形成了完整闭环。
