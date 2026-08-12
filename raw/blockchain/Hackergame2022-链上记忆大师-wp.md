# 链上记忆大师

## 题目简述

服务端要求提交一段 EVM 合约字节码。测试合约先调用 `memorize(n)`，再调用 `recall()`，并检查返回值是否等于随机数 $n$。三问依次禁止正常存储、回滚状态修改，并最终把 `memorize` 限制为只读调用，需要寻找 EVM 中仍能跨调用保留的信息。

## 解题过程

### 记忆练习：直接使用 Storage

第一问没有额外限制，使用状态变量即可：

```solidity
pragma solidity =0.8.17;

contract MemoryMaster {
    uint256 private saved;

    function memorize(uint256 n) external {
        saved = n;
    }

    function recall() external view returns (uint256) {
        return saved;
    }
}
```

用 Solidity 0.8.17 编译，提交部署字节码，合约即可通过所有测试。

### 牛刀小试：用 gas 消耗传递 16 位信息

第二问把 `memorize(n)` 放在一个最终会 `revert` 的子调用中。Storage、余额等状态修改都会回滚，但已经消耗的 gas 不会返还。因此可让 `memorize` 按 $n$ 的大小故意消耗不同 gas，再由 `recall` 读取 `gasleft()` 反推 $n$。

在 50000000 的调用 gas 上限内，需要区分 $2^{16}=65536$ 个值，每档约有 763 gas。官方方案取每档 700 gas：

```solidity
function memorize(uint16 n) external view {
    uint256 start = gasleft();
    while (gasleft() > start - uint256(n) * 700) {
        gasleft();
    }
}
```

进入外部调用时，EVM 的 EIP-150 规则最多只传递剩余 gas 的 $63/64$；函数分派和固定指令也有常量开销。因此先让 `recall` 直接 `revert` 并携带 `gasleft()`，用一组已知 $n$ 标定常数。官方编译设置下，测得偏移约为 30384：

```solidity
function recall() external view returns (uint16) {
    uint256 left = gasleft();
    return uint16(
        (50000000 - 30384 - left / 63 * 64 + 350) / 700
    );
}
```

`+350` 用于四舍五入。编译器版本、优化参数变化会改变固定开销，所以应以自己最终提交的字节码重新标定，而不是照搬常数。

### 终极挑战：把地址冷热状态当作 256 位缓存

第三问要求两个函数都是 `view`，测试时会通过 `STATICCALL` 调用，无法修改 Storage、余额或创建合约。单一 gas 数值也不足以可靠编码 256 位整数。

EIP-2929 为同一笔顶层调用维护“已访问地址”集合：第一次访问某地址是冷访问，开销约 2600 gas；同一交易内再次访问则为热访问，约 100 gas。这个集合能跨越 `memorize` 和 `recall` 的两个内部调用，而且读取地址信息在 `STATICCALL` 中合法。

用地址 `0x100+i` 表示第 $i$ 位。`memorize` 只访问值为 1 的位：

```solidity
uint256 constant BASE = 0x100;

function memorize(uint256 n) external view {
    for (uint256 i = 0; i < 256; i++) {
        if (((n >> i) & 1) != 0) {
            uint256 ignored = address(uint160(BASE + i)).balance;
        }
    }
}
```

`recall` 再逐个访问同一批地址，并测量单次访问前后的 gas 差。小于 1000 说明地址已经热过，对应位为 1：

```solidity
function recall() external view returns (uint256 n) {
    for (uint256 i = 0; i < 256; i++) {
        uint256 beforeGas = gasleft();
        uint256 ignored = address(uint160(BASE + i)).balance;
        if (beforeGas - gasleft() < 1000) {
            n |= uint256(1) << i;
        }
    }
}
```

起始地址选 `0x100`，是为了避开交易开始时就被视为热地址的预编译合约区域。编译时还要确认优化器没有删掉看似未使用的余额读取；可以关闭优化，或检查生成字节码中对应的 `BALANCE` 指令仍然存在。

这条信道不能直接用于第二问，因为发生回滚时，访问列表的冷热变化也会随子调用一起回滚；而第三问没有回滚，两个调用都处于同一次 `test` 执行中，所以状态得以保留。

## 方法总结

EVM 中“状态”不只包含 Storage。gas 消耗不会因 revert 返还，地址访问列表则在同一交易中形成临时缓存。第一问使用持久状态，第二问使用不可逆的资源消耗，第三问使用交易级微架构状态。解题时应先明确每种状态的生命周期，以及 `CALL`、`STATICCALL`、revert 对它们分别有什么影响。
