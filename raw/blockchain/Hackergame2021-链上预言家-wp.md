# 链上预言家

## 题目简述

题目要求预测挑战合约即将创建的子合约地址。第一关使用公开盐值和 `CREATE2`，地址可由部署者、salt 与初始化代码预先计算；第二关把 seed 藏在挑战合约内部，但允许调用创建函数。关键是利用 EVM 的回滚语义：内层创建得到地址后让外层调用主动 `revert`，既带出地址，又撤销对子合约地址 nonce/代码状态的占用。

## 解题过程

### 第一关：直接计算 CREATE2 地址

EIP-1014 定义的地址公式是：

$$
A=\operatorname{keccak256}(\texttt{0xff}\Vert D\Vert S\Vert
\operatorname{keccak256}(I))[12:]
$$

其中 $D$ 是部署合约地址，$S$ 是 32 字节 salt，$I$ 是 init code。Solidity 中可以直接复现：

```solidity
function predict(
    address deployer,
    bytes32 salt,
    bytes memory initCode
) pure returns (address) {
    bytes32 h = keccak256(
        abi.encodePacked(
            bytes1(0xff),
            deployer,
            salt,
            keccak256(initCode)
        )
    );
    return address(uint160(uint256(h)));
}
```

`initCode` 必须与题目部署 `Dummy` 时的字节完全一致，包含编译器生成的构造代码和 metadata；仅复制 runtime bytecode 会算出错误地址。把计算结果提交给挑战合约即可得到第一关 flag，固定语义前缀为 `flag{CRE4TE2_c0ntr4ct_Addr_1s_pr3d1ct4ble_...}`。

### 第二关：用 revert 读取并回滚地址

第二关的 seed 无法从另一个合约直接读取，但挑战合约提供 `create_child()`，调用后会返回新地址。若正常调用，第一次创建已经改变链上状态，随后预测值就不是“下一个”地址。

解决方法是在预测合约里做一次外部自调用，并在辅助函数拿到地址后立刻把它编码进 revert data：

```solidity
contract Predictor {
    IChallenge challenge;

    function probe() external {
        require(msg.sender == address(this));
        address child = challenge.create_child();
        revert(string(abi.encode(child)));
    }

    function predictByRollback() external returns (address result) {
        try this.probe() {
            revert("unexpected success");
        } catch Error(string memory data) {
            result = abi.decode(bytes(data), (address));
        }
    }
}
```

`probe` 的回滚会连带撤销其内部 `create_child()` 的全部状态变化，但 revert data 仍能被上一层 `try/catch` 捕获。捕获到的地址因此既是真实创建结果，又仍然是挑战合约下一次创建会使用的地址。提交该值完成第二关。

## 方法总结

- 核心技巧：第一关精确复现 `CREATE2` 地址公式；第二关用嵌套调用、revert data 和事务回滚实现无状态探测。
- 识别信号：题目要求预测合约地址，同时提供创建函数或可捕获的外部调用，而正常探测会污染状态。
- 复用要点：区分 init code 与 runtime code；EVM 回滚会撤销内部调用的状态，却不会阻止上层捕获返回的错误数据。
