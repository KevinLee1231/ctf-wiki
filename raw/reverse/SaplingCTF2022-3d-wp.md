# 3D

## 题目简述

程序把一个 $8\times8\times8$ 三维迷宫编码在二进制中，接受 w、a、s、d、q、e 六个方向的移动序列。到达终点后，程序以整个路径的 SHA-256 作为 AES-256-CBC 密钥解密 flag。需要先恢复迷宫连边，再求一条有效路径。

## 解题过程

每个格子的低若干位分别表示六个方向是否可走。按源码坐标核对，起点是 $(7,1,0)$，终点是 $(7,0,7)$；官方说明文字曾把两者写反，但实际校验代码和 solver 均使用这一方向。

把每个格子作为图节点，按位标志添加相邻边，再做 BFS：

~~~python
from collections import deque

queue = deque([(start, "")])
seen = {start}
while queue:
    pos, path = queue.popleft()
    if pos == target:
        password = path
        break
    for key, nxt in neighbours(pos, maze[pos]):
        if nxt not in seen:
            seen.add(nxt)
            queue.append((nxt, path + key))
~~~

最短路径为：

~~~text
sqasasaaawwqqqqdqqwdedddwq
~~~

程序计算 SHA-256(path)，以固定 IV 0123456789012345 做 AES-CBC 解密：

~~~python
from hashlib import sha256
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = sha256(password.encode()).digest()
plain = AES.new(
    key,
    AES.MODE_CBC,
    b"0123456789012345",
).decrypt(ciphertext)
print(unpad(plain, 16))
~~~

结果为：

~~~text
maple{aMAZE1ng_job!11!1}
~~~

## 方法总结

迷宫逆向应先把程序内部表示转换为明确的图，再交给 BFS/DFS，而不是手工试路。最终解密密钥若来自路径字符串，方向字符、起终点和编码顺序必须与程序完全一致。遇到官方文字与源码冲突，应以实际校验逻辑和正向解密结果为准。
