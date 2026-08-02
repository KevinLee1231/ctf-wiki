# teenage-game

## 题目简述

玩家从二维地图 $(4,4)$ 出发，`w/a/s/d` 直接增减行列，没有任何边界检查；`l` 还能把全局 `player_tile` 改成任意下一字节。每次移动都会把新位置写成该 tile，因此可以把地图越界索引变成单字节任意值写。

发布二进制启用了 PIE、canary、NX 与 Full RELRO，但这个定点单字节写不需要破坏 canary。PIE 页内低位固定，改写返回地址最低字节即可跳到同一映像中的 `win`。

## 解题过程

主函数中地图基址为 `rbp-0xa90`，调用 `move_player` 时当前返回地址位于 `rbp-0xaa8`。二维索引

$$
\text{map}[-1][66]=\text{map}+(-90+66)=\text{map}-24
$$

恰好落在该调用的返回地址最低字节。

先用 `l` 把 tile 设为 `win+5` 的低字节，再从列 4 向右 62 次到列 66，最后向上 5 次让行从 4 变成 $-1$：

```python
from pwn import ELF

exe = ELF("./bin/game", checksec=False)
payload = bytearray(b"lX" + b"d" * 62 + b"w" * 5)
payload[1] = (exe.sym["win"] + 5) & 0xff
```

最后一次写覆盖 `move_player` 的返回地址低字节，函数返回到 `win+5`，跳过常规序言并调用 `/bin/sh`。本地以同一 payload 实际复验后可直接执行命令，读取到：

```text
tjctf{so_many_new_features_but_who_will_stop_the_underflow?_47c6f204377cb18b30e68da46e9930dc}
```

## 方法总结

- 二维数组下溢同样是内存写原语；应把 $(r,c)$ 化成线性偏移，检查它与当前栈帧敏感槽位的关系。
- PIE 不会随机化页内偏移，在原返回地址与目标函数位于兼容范围时，单字节覆盖仍可改变控制流。
- 这条链通过精确坐标绕过 canary，说明编译保护不能替代对每次移动后的行列边界检查。
