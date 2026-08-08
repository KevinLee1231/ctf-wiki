# MiniLCTF2020 - machine

## 题目简述

网页加载经过两轮 JavaScript 混淆的脚本，并把 flag 按连字符分段后做 Base64、逆序和逐字符异或。既可以在开发者工具中对所有 `return` 下断点读取运行时局部变量，也可以从仓库保留的未混淆 `index.js` 直接写逆变换。

## 解题过程

未混淆函数对每段执行：

```javascript
function encode(x) {
    let a = Buffer.from(x, 'utf8').toString('base64');
    a = a.split('').reverse().join('');
    let b = [];
    for (let i = 0; i < a.length; i++) {
        b.push(String.fromCharCode(a.charCodeAt(i) ^ i));
    }
    return b.join('');
}
```

因此逆过程是先再次按位置异或，再逆序，最后 Base64 解码：

```python
import base64

def decode_part(part: str) -> str:
    reversed_b64 = ''.join(chr(ord(ch) ^ i) for i, ch in enumerate(part))
    b64 = reversed_b64[::-1]
    return base64.b64decode(b64).decode()

parts = [...]  # 页面显示的四段运行时字符串
print('-'.join(decode_part(x) for x in parts))
```

部署脚本经过双重 AST 改名和结构混淆，复制含控制字符的页面文本不方便时，打开 DevTools 的 Sources，格式化 `index.js`，在各个 `return` 处下断点。定时回调最终会把实际 flag 传给处理函数；在返回字符串前检查局部变量即可直接看到完整 `minil{...}`。

参赛截图中的 UUID flag 属于当时动态实例。当前源码只保留占位符 `flag_here`，没有足够证据复原已下线实例的固定字符串，因此这里不照抄旧 UUID 作为可复现结果。

## 方法总结

JavaScript 混淆不等于加密。优先识别稳定的数据变换，再决定静态逆运算还是运行时断点。遇到控制字符输出时，在变换函数内部观察解码前后的局部变量通常比从 DOM 复制更可靠。
