# 链上转账助手

## 题目简述

题目要求提交一段 EVM 合约创建字节码。服务会用这段字节码部署 10 个接收合约，再让批量转账合约分别向它们转入 1 ether；只有整笔 `batchTransfer` 交易最终回滚，服务才会给出对应小题的 flag。

三问逐步收紧批量转账逻辑：

- 第一问用 Solidity 的 `transfer` 逐个转账，任意接收方回滚都会让外层交易一起失败。
- 第二问改用低级 `call`，单次失败只会被记录到 `pendingWithdrawals`，循环仍会继续。
- 第三问继续使用低级 `call`，但把每个接收方的显式 gas 限制为 10000，试图阻止接收合约拖垮调用者。

决定解法的是 EVM Message Call 的失败传播、EIP-150 的 gas 转发规则，以及 Solidity 对未使用返回数据仍会自动复制这一行为。虽然题目通过网络服务接收字节码，核心障碍是智能合约调用与 gas 语义，因此归入 `blockchain`。

## 解题过程

### 服务的判定条件

服务端会连续部署 10 份相同的玩家合约，然后以 1000000 gas 调用：

```python
amounts = [w3.to_wei(1, "ether")] * 10
tx = challenge.functions.batchTransfer(recipients, amounts).build_transaction({
    "from": acct.address,
    "value": sum(amounts),
    "gas": 1000000,
})

if tx_receipt.status:
    print("Transfer success, no flag.")
else:
    print(open(f"flag{challenge_id}").read())
```

因此目标不是偷走转账金额，而是针对三种批量转账实现构造能够使外层交易失败的接收合约。将下面各问的 Solidity 合约用附件中的 `compile.py` 或 Remix 编译，提交创建字节码即可验证。

### 第一问：在 `receive` 中主动回滚

第一版循环直接调用：

```solidity
recipients[i].transfer(amounts[i]);
```

`transfer` 失败会抛出异常，外层没有捕获，整个 `batchTransfer` 随即回滚。接收合约只需在收到空 calldata 的普通转账时执行 `revert`：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Receiver {
    receive() external payable {
        revert();
    }
}
```

甚至一个没有 `receive` 或 payable fallback 的空合约也会拒绝普通转账，但显式写出 `revert` 更能说明利用条件。

### 第二问：让批量调用耗尽 gas

第二版改成低级调用，并在失败时记账：

```solidity
(bool success, ) = recipients[i].call{value: amounts[i]}("");
if (!success) {
    pendingWithdrawals[recipients[i]] += amounts[i];
}
```

单纯回滚只能让当前接收方进入 `pendingWithdrawals`，不能使外层交易失败。低级调用没有显式 gas 上限；按照 EIP-150，调用者最多把剩余 gas 的 $63/64$ 交给被调用者。接收合约可以持续执行，尽量耗尽获配 gas：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Receiver {
    receive() external payable {
        while (gasleft() > 100) {}
    }
}
```

一次内部调用耗尽 gas 后，外层仍会留下约 $1/64$ 的 gas；但题目连续调用 10 个恶意接收方，每轮还要承担 `CALL`、失败分支和存储写入的固定开销，剩余 gas 很快降到不足以继续执行，最终使整笔交易 out-of-gas。

### 第三问：return bomb

第三版把内部调用改为：

```solidity
(bool success, ) = recipients[i].call{value: amounts[i], gas: 10000}("");
```

普通死循环只能烧掉接收方获配的有限 gas，调用者仍能处理失败并继续。突破口是返回数据：Solidity 即使没有使用低级调用的第二个返回值，生成的代码仍会把完整 returndata 复制进调用者内存。被调用合约扩张的是自己的内存，而返回后复制和再次扩张内存的成本由批量转账合约承担，这就是 return bomb。

EVM 内存成本可写成：

$$
w=\left\lceil\frac{n}{32}\right\rceil,\qquad
C_{\mathrm{mem}}(w)=3w+\left\lfloor\frac{w^2}{512}\right\rfloor,
$$

其中 $n$ 是使用到的内存字节数。接收合约获得显式的 10000 gas；带 value 的 `CALL` 还会给接收方 2300 gas stipend，总计约 12300 gas。反推内存扩张成本并留出指令余量，官方解法选择返回 59200 字节：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Receiver {
    receive() external payable {
        assembly {
            return(0, 59200)
        }
    }
}
```

`return(0, 59200)` 会先把接收合约内存扩张到相应大小，再把这段 returndata 暴露给调用者。单个接收合约没有耗尽自己的 12300 gas，但调用者在 10 轮中反复复制大块返回数据并承担二次增长的内存成本，最终耗尽整笔交易的 gas。把 `return` 换成 `revert` 也能构造大份失败数据，并会让外层再执行失败记账，不过本题无需这点额外开销即可通过。

实际合约若要防御此类问题，不能仅限制被调用方 gas，还要限制复制的返回数据长度。Nomad 的 [`ExcessivelySafeCall`](https://github.com/nomad-xyz/ExcessivelySafeCall) 就是在底层调用后只复制调用者允许的固定字节数，避免任意大 returndata 消耗调用者 gas。

## 方法总结

- 核心技巧：第一问利用 `transfer` 的失败传播；第二问利用多轮 EIP-150 gas 消耗；第三问利用未受限 returndata 复制制造 return bomb。
- 识别信号：看到批量外部调用、忽略低级调用返回数据、只限制 callee gas 却不限制 returndata 长度时，应同时检查失败分支、内存扩张和返回数据复制成本。
- 复用要点：外部调用的 gas 限额只约束被调用者执行，不能自动约束调用者处理返回数据的成本；安全封装应同时限制 gas、检查成功状态，并为 returndata 设置明确上限。
