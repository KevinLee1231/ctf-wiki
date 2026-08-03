# UIUCTF 2023 Geoguesser Writeup

## 题目简述

题目给出 Janet 解释器和编译后的 `program.jimage`，远程服务会生成一组随机经纬度，选手有 5 次机会输入答案。坐标比较精度为 $10^{-4}$，直接猜中几乎不可能。

决定性问题是随机数发生器的种子：程序用当前 Unix 时间初始化 Janet 的 `math/rng`。只要恢复字节码逻辑，并在本地用相近时间和相同 PRNG 生成坐标，就能预测服务端答案。

## 解题过程

Janet 可以直接载入 image 并列出其中的定义：

```janet
(load-image (slurp "program.jimage"))
```

输出中可以看到 `init-rng`、`random-float`、`guessing-game`、`compare-coord` 和 `print-flag` 等函数及源码位置。还原后的关键逻辑等价于：

```janet
(defn init-rng []
  (set rng (math/rng (os/time))))

(defn random-float [min max]
  (+ min (* (math/rng-uniform rng) (- max min))))

(def answer [(random-float -90 90)
             (random-float -180 180)])
```

纬度由 $[-90,90)$ 映射得到，经度由 $[-180,180)$ 映射得到。服务端和本地脚本都使用 `os/time` 作为种子，因此只需复现相同调用顺序。网络连接、进程启动和秒级时钟边界可能造成少量偏差；题目允许 5 次猜测，枚举本地当前时间附近的几个种子即可。

预测脚本可写为：

```janet
(defn random-float [rng min max]
  (+ min (* (math/rng-uniform rng) (- max min))))

(defn main [&]
  (def now (os/time))
  (loop [delta :range [-1 2]]
    (def rng (math/rng (+ now delta)))
    (def lat (random-float rng -90 90))
    (def lon (random-float rng -180 180))
    (printf "%.5f,%.5f" lat lon)))
```

这里依次尝试 `now - 1`、`now`、`now + 1`。保留 5 位小数足以通过严格小于 $10^{-4}$ 的比较。连接服务后，把三组候选坐标逐行提交，读到 `You win!` 即停止：

```python
from pwn import process, remote

io = remote("geoguesser.chal.uiuc.tf", 1337)
gen = process(["./janet", "solve.janet"])
answers = gen.recvall().decode().strip().splitlines()

for answer in answers:
    io.sendlineafter(b"Where am I? ", answer.encode())
    if b"You win!" in io.recvline():
        break

io.recvuntil(b"uiuctf{")
print((b"uiuctf{" + io.recvuntil(b"}")).decode())
```

最终得到：

```text
uiuctf{i_se3_y0uv3_f0und_7h3_t1m3_t0_r3v_th15_b333b674c1365966}
```

## 方法总结

这道题把语言字节码逆向与时间种子预测结合在一起。遇到不熟悉的 image/bytecode 格式时，应先利用语言运行时自带的加载、反汇编或元数据接口恢复高层函数；确认 PRNG 类型、种子、调用次数和数值映射后，再用同一运行时复现，能避免不同语言随机数算法不兼容。对秒级时间种子，围绕当前时间做小范围容错枚举通常比追求完全同步更稳妥。
