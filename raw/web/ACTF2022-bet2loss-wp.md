# bet2loss

## 题目简述

题目由一个 `BetToken` 合约和配套 Web 服务组成。玩家先通过 `airdrop()` 获得 30 个 token，再调用 `bet(value, mod)` 猜测下面的伪随机数：

```solidity
uint256 rand = uint256(
    keccak256(
        abi.encodePacked(nonce, block.timestamp, block.difficulty, msg.sender)
    )
) % mod;
```

每次下注消耗 10 个 token，`mod` 被限制在 2 到 12，合约还用 `extcodesize(msg.sender) == 0` 阻止普通合约参与。玩家地址余额超过 2000 后，Web 服务会用庄家私钥代签 `checkWin(player)`；交易成功才返回 flag。

真正的攻击面不只在链上随机数。Web 服务把庄家 keystore、密码和下注日志放在同一数据目录，并暴露了 `/log` 下载接口；仓库同时给出了利用构造期代码长度为零的纯链上替代解。

## 解题过程

### 预期解：通过 `/log` 读取庄家密钥

服务启动时从数据目录读取 `keystore.txt` 和 `password.txt`，恢复庄家私钥，然后用该账户部署合约、调用 `checkWin`。问题出在 `/log`：

```javascript
address = req.body.address;
filename = address + ".txt";
filepath = path.join(datadir, filename);

var files = fs.readdirSync(datadir);
if (files.indexOf(filename) !== -1) {
    res.download(filepath);
}
```

接口没有验证 `address` 是否为以太坊地址，只要拼出的文件名真实存在就会下载。因此分别提交 `address=password` 和 `address=keystore`，即可取得密码与加密 keystore：

```python
import requests

password = requests.post(BASE + "/log", json={"address": "password"}).content
keystore = requests.post(BASE + "/log", json={"address": "keystore"}).json()
```

用密码解开 keystore 后得到庄家账户。随后以庄家身份调用 `seal(player, 2001)`，令玩家余额满足 `balances[player] > 2000`。最后向 `/flag` 提交玩家地址；服务端代签并发送 `checkWin(player)`，交易状态为成功时返回 flag。

这里不需要攻击 `checkWin` 本身：关键是 Web 层把链上 owner 权限所依赖的密钥文件暴露给了用户。

### 纯链上替代解：构造期绕过与地址复用

`bet()` 通过 `EXTCODESIZE` 拒绝合约地址，但合约构造函数执行期间，运行时代码尚未写入账户，`extcodesize(address(this))` 为零。官方替代解让临时 `Callee` 在构造函数中直接调用 `airdrop()` 或 `bet()`，从而绕过检查。

下注前可读取 `BetToken` 的 `nonce` 存储槽，并在同一笔交易的构造函数中使用当前区块的 `timestamp`、`difficulty` 和预先确定的 `CREATE2` 地址计算正确的 `rand`。每次成功下注先扣 10，再增加 `10 * 12`，净增 110 个 token。

临时合约完成调用后执行 `selfdestruct`。在本题使用的旧 EVM 语义下，用相同部署者、salt 和初始化代码再次执行 `CREATE2`，可以在同一地址重新部署；`BetToken` 中该地址的余额和下注次数仍然保留。每个新区块重复一次，直到该地址余额超过 2000，再把该地址提交给 `/flag`。

仓库还记录了一条非预期路线：比赛链的出块间隔稳定，使 `block.timestamp` 具有可预测性。它同样削弱了随机数，但不如密钥文件泄露路线直接。

## 方法总结

- 核心技巧：利用下载接口缺少语义校验，读取 keystore 与密码后接管合约 owner 权限。
- 识别信号：链上关键操作由 Web 后端代签，而后端同时提供日志或文件下载接口时，应检查密钥材料是否与用户可控文件名共用目录。
- 复用要点：`EXTCODESIZE` 不能可靠区分 EOA 与合约，构造期代码长度为零；旧 EVM 中 `CREATE2 + SELFDESTRUCT` 还可能让同一地址反复执行构造逻辑。
