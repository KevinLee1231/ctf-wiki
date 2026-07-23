# rainbet

## 题目简述

服务端交替生成扫雷和过马路两种赌博游戏，连续 25 次取得“最大胜利”才返回 flag。每局内容由 WebAssembly 中的自定义伪随机生成器根据 `session_id` 和当前 streak 决定。

WebSocket 消息还要求附带当前画面的 HMAC，但 `/api/sessioninfo` 会把本会话签名密钥提供给前端。因此真正的障碍是还原 WASM 随机算法并准确预测每局。

## 解题过程

逆向 `rainbet_gen.wasm` 可恢复：

- 32 字节固定 `SECRET`；
- 8 个 64 位初始状态；
- 16 个轮常量和 256 字节 S 盒；
- 三个域分隔常量；
- 16 轮 ARX、SubBytes、lane mixing 与 lane permutation。

初始化时依次吸收固定 secret、ASCII `session_id` 和 `round_idx`，每个阶段加入不同域分隔常量并执行置换。输出生成器每取完 8 个 lane 就再次置换。

若首个随机数最低位为 0，则生成扫雷：

1. 网格大小从 5、6、7、8 中选择；
2. 计算雷数；
3. 用同一随机流对位置数组做前缀 Fisher-Yates；
4. 雷区就是洗牌后的前 `num_mines` 项。

否则生成 24 步过马路游戏；每一步把随机 64 位数与阈值 `7378697629483820646` 比较，决定该位置是否有车。

客户端按预测结果行动：

- 扫雷：依次翻开所有不在雷集合中的格子；
- 过马路：走到第一辆车之前，随后立即 `cashout`；若没有车则走满 24 步。

每次动作前，用 `/api/sessioninfo` 返回的 secret 对服务端规定的规范字符串做 HMAC-SHA256。状态更新后必须使用新的 `revealed` 或 `crossed` 值重新签名，不能复用旧 token。

连续预测并完成 25 局后得到：

```text
UMDCTF{one_might_argue_that_gambling_is_the_best_vice_but_they_would_be_wrong}
```

## 方法总结

题目把随机逻辑放进 WASM 只能增加阅读成本，并没有隐藏客户端可获得的确定性输入。完整复刻置换、字节序和取样顺序后，`session_id + streak` 足以预测每局。协议层还需同步维护规范视图并逐动作重签 HMAC，否则即使游戏预测正确也会被拒绝。
