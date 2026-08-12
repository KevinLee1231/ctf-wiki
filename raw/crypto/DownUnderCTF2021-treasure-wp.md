# DownUnderCTF 2021 - treasure

## 题目简述

服务把秘密坐标 $s$ 拆成三份。对素数 $p$ 和随机数 $r_1,r_2$，生成：

$$
s_1=r_1r_2s,\qquad
s_2=r_1^2r_2s,\qquad
s_3=r_1r_2^2s\pmod p.
$$

组合器按
$s=s_1^3(s_2s_3)^{-1}\bmod p$
恢复秘密。攻击者控制第一份 share：第一次可故意输入错误值并看到组合结果，第二次必须让结果等于指定的假坐标，最后还要提交真实坐标。该方案没有验证 share 是否由同一轮分发生成，因此组合器本身构成代数 oracle。

## 解题过程

第一次提交 $s_1'=1$。组合器返回：

$$
v=(s_2s_3)^{-1}\bmod p.
$$

服务会容忍这一次结果不是坐标。有了 $v$，要让下一次组合结果等于公开目标 $s_{fake}$，只需解：

$$
(s_1')^3v=s_{fake}\pmod p,
$$

即：

$$
s_1'=\sqrt[3]{s_{fake}v^{-1}}\pmod p.
$$

Sage 可以直接在 $\mathbb Z_p$ 中求模立方根：

```sage
FAKE_COORDS = 5754622710042474278449745314387128858128432138153608237186776198754180710586599008803960884
p = 13318541149847924181059947781626944578116183244453569385428199356433634355570023190293317369383937332224209312035684840187128538690152423242800697049469987

F = Zmod(p)

real_share = Integer(receive_initial_share())
send_share(1)
inverse_product = F(receive_revealed_value())

fake_share = F(FAKE_COORDS / inverse_product).nth_root(3)
send_share(Integer(fake_share))
```

这里 `FAKE_COORDS / inverse_product` 是域除法，等价于
$s_{fake}s_2s_3$。目标值由题目预先选择，保证所需立方根存在。

真实坐标无需恢复 $r_1,r_2$。使用服务器最初给出的真实 $s_1$ 与第一次泄露的 $v$：

$$
s_{real}=s_1^3v\pmod p.
$$

```sage
real_coords = F(real_share)^3 * inverse_product
send_real_coords(Integer(real_coords))
```

最终得到：

```text
DUCTF{m4yb3_th3_r34L_tr34sur3_w4s_th3_fr13nDs_w3_m4d3_al0ng_Th3_W4y.......}
```

## 方法总结

本题不是门限不足，而是 share 不可验证且组合结果会回显。攻击者用一次选择 share 查询得到朋友两份 share 乘积的逆元，随后既能控制组合输出，也能恢复真实秘密。设计秘密共享协议时不能只保证代数上可重构，还必须验证参与者提交的 share 与原始承诺一致，并避免向恶意参与者回显可重复利用的中间组合结果。
