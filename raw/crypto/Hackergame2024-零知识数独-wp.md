# 零知识数独

## 题目简述

题目提供 9×9 数独的 Circom 电路、WASM witness 生成器、Groth16 proving key 和 verification key。公开输入是未完成数独，秘密 witness 是完整解；服务端用固定公开题面和 verification key 检查 `proof.json`。

前两问分别是普通数独求解和按给定电路生成 Groth16 证明。第三问的公开题面实际无解，需要利用 R1CS 约束缺失：正数检查用 `<--` 只计算 witness、没有把比较关系写入约束；同时 `LessEqThan(4)` 在有限域边界上接受若干负数。决定性主障碍是 ZK 电路约束安全而非链上状态，归为 `crypto`。

## 解题过程

### 数独高手与 ZK 高手

第一问可直接对前端给出的 81 个格子做回溯：选择候选最少的空格，依次尝试 1 到 9，并维护行、列和 3×3 宫占用集合。前端数据篡改也能完成第一问，但不帮助理解后面的证明。

第二问先求出 expert 数独的合法解，再按电路输入名组织 JSON：

```text
{
  "unsolved_grid": [81 个公开格子],
  "solved_grid": [81 个秘密解格子]
}
```

用题目给出的 WASM 和 zkey 生成证明，并用 verification key 本地验证：

```bash
snarkjs groth16 fullprove input.json sudoku.wasm sudoku.zkey proof.json public.json
snarkjs groth16 verify verification_key.json public.json proof.json
```

通过后向服务端提交 `proof`、`publicSignals` 和难度字段。Groth16 的公开输入顺序必须与电路导出的 `public.json` 一致，不能自行重排 81 个格子。

### 无解之谜：找出 R1CS 中不存在的检查

Circom 最终编译成

$$
Az\circ Bz=Cz,
$$

其中 $z=1\parallel x\parallel w$，$x$ 是公开输入，$w$ 是秘密 witness。只有约束运算符生成的关系会进入 $A,B,C$；普通 witness 赋值 `<--` 不生成约束。

原电路写成：

```circom
gt_zero_signals[i][j] <-- (solved_grid[i][j] > 0);
gt_zero_signals[i][j] === 1;
```

第二行只约束 `gt_zero_signals` 等于 1，却没有约束它必须等于比较表达式的结果。因此可以复制电路并仅修改 witness 计算提示：

```circom
gt_zero_signals[i][j] <-- 1;
gt_zero_signals[i][j] === 1;
```

这会改变 WASM 如何填写 witness，但不会改变 R1CS。故可以用修改后电路生成 witness，同时继续使用题目原来的 zkey 和 verification key；如果修改了任何真正的约束，则 proving key 将不再匹配。

### 利用有限域范围检查

剩余范围检查是：

```circom
component upperBound = LessEqThan(4);
upperBound.in[0] <== value;
upperBound.in[1] <== 9;
```

`LessEqThan(4)` 本质上对 `value + 2^4 - 9` 做 5 bit 分解并检查最高位。在素域 $F_p$ 中，$p-k$ 表示 $-k$；这个构造实际接受 $[-6,9]$，而不是预期的 $[0,9]$。行、列、宫的 `SudokuChecker` 只要求九个值两两不同且不等于 0，因此可以把符号集合从 1 到 9 扩展为

$$
\{-6,-5,-4,-3,-2,-1,1,2,\ldots,9\}.
$$

一组满足无解题面公开格、互异关系和错误范围约束的 witness 为：

```text
 9  6 -3 | 8  3  4 | 1  5 -4
 8  4 -4 | 1  5  6 | 2 -1 -2
 7 -1 -2 | 2  9 -4 | 3  4 -3
---------+---------+---------
 4  7  1 | 3  2  5 | 8  9  6
 5  2  8 | 6  1  9 | 4  7  3
 6  9  3 | 4  7  8 | 5  1  2
---------+---------+---------
-4  1  4 | 5  8  2 |-1  6  9
-1  5  2 | 9  4  3 |-4  8  7
-2  3  9 | 7  6  1 |-3  2  4
```

在修改后的电路上只重新编译 witness 生成器，然后生成证明：

```bash
circom sudoku.circom --r1cs --sym --wasm --O2 -o groth16
node groth16/sudoku_js/generate_witness.js \
  groth16/sudoku_js/sudoku.wasm impossible.json witness.wtns
snarkjs groth16 prove sudoku.zkey witness.wtns proof.json public.json
snarkjs groth16 verify verification_key.json public.json proof.json
```

安全检查是比较原版与修改版编译出的 R1CS 约束是否一致；本题修改的只是 `<--` 提示，所以证明仍会被原 verification key 接受。随后将 `public.json` 和 `proof.json` 提交给 impossible 难度接口即可。

本次整理完整核对了原始 `sudoku.circom`、修改版电路、恶意 witness、证明生成脚本和公开信号。没有重装 Node/Circom 环境，也没有重新执行可信设置或远程提交；官方目录中的现成 `proof.json` 可由同目录 verification key 静态对应，但当前未现场运行 `snarkjs verify`。

## 方法总结

- 核心技巧：区分 Circom 的 witness 计算提示与真正 R1CS 约束；修改未受约束提示生成恶意 witness，再利用有限域比较器的边界扩大可用数值集合。
- 识别信号：电路中出现 `<--`、比较结果被赋给中间 signal、只做单侧范围检查或直接把整数语义搬到有限域时，应立即审计 under-constrained 与 bit-length 问题。
- 复用要点：proving key 绑定 R1CS，不绑定 witness 生成代码；攻击时只能改提示而不能改约束。范围证明必须同时约束上下界和位长，并对负数的域表示做明确处理。
