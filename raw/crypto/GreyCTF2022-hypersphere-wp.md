# GreyCTF2022 - HyperSphere

## 题目简述

题目在四维单位超球面上定义类似四元数乘法的群运算，并用 $A=g^a$、$B=g^b$、$g^{ab}$ 构造密钥交换。难点是把看似特殊的点运算转化成有限域上的线性代数与离散对数。

## 解题过程

源码中的 `Point` 乘法等价于四元数左乘。固定生成元 $g=(w,x,y,z)$ 后，可写出对应的 $4\times4$ 乘法矩阵 $M_g$，使点乘幂等价于矩阵幂：

$$g^a\longleftrightarrow M_g^a.$$

攻击者可以提交自己选择的参数。选择便于对角化的 $g$ 后，对 $M_g$ 求 Jordan 分解；在扩域中，矩阵幂转化为特征值的标量幂。比较 $A$ 的对应特征值即可求离散对数 $a$，再计算共享点 $B^a$。

```sage
M = multiplication_matrix(g)
J, P = M.jordan_form(transformation=True)
# 从 J 与 A 的特征值分量求 discrete_log
a = discrete_log(lambda_A, lambda_g)
shared = B ** a
```

服务使用共享点序列化结果派生 SHAKE 密钥流并异或 flag。按同样顺序编码坐标、生成密钥流并解密，得到：

```text
grey{HyperSphereCanBeUsedForKeyExchangeToo!(JustProbablyNotThatSecure)_33JxCZjzQQ7dVGvT}
```

## 方法总结

遇到自定义群时，应先识别它是否只是熟悉代数结构的换皮。本题的四维乘法可以忠实表示为矩阵，群上的离散对数因而降为特征值所在有限域中的离散对数。实现时需保持坐标顺序、扩域和密钥序列化完全一致。
