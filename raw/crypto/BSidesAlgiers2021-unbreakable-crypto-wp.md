# Unbreakable_crypto

## 题目简述

应用把 JSON 票据使用随机 IV 和固定 16 字节密钥做 AES-CBC 加密，返回“Base64 编码的十六进制 IV‖密文”。用户只能控制 name 字段，但可以反复取得已知明文对应的 token。解密后，服务用宽松的字符串切分构造 Ticket 对象，再让带 flag Cookie 的管理员浏览器访问验证页面。

验证模板把票据 type 直接放进未加引号的 img 元素 alt 属性。只要把解密后的 type 改成 HTML 事件属性，就能形成 XSS。决定性步骤是跨多个分组的 CBC bit flipping，因此本题归入 crypto，而不是按外层 Flask 应用机械归入 web。

## 解题过程

CBC 解密满足：

$$
P_i=D_K(C_i)\oplus C_{i-1}.
$$

已知原明文块 $P_i$ 时，要把它改成 $P_i'$，只需令：

$$
C_{i-1}'=C_{i-1}\oplus P_i\oplus P_i'.
$$

其中 $C_0$ 就是 IV。对应实现如下：

~~~python
from Crypto.Util.strxor import strxor

def forge_previous(previous, known_plain, wanted_plain):
    assert len(previous) == 16
    assert len(known_plain) == 16
    assert len(wanted_plain) == 16
    return strxor(
        strxor(previous, known_plain),
        wanted_plain,
    )
~~~

应用允许选择任意长度的 name，因此先请求大量字符“A”，就能准确构造完整 JSON 及 PKCS#7 padding。然后把已知明文和目标明文都切成 16 字节块，对需要控制的块修改其前一密文块。

修改 $C_{i-1}$ 会让 $P_{i-1}$ 整块变成不可预测数据。官方解法因此交替安排“可牺牲垃圾块”和“受控脚本块”，只在后者中放入 HTML 属性与分段 Base64 JavaScript；宽松的 Ticket.parse 会删除引号、单引号和换行，却不会阻止 type 中出现事件处理器。最终 type 可形成等价于下面的结构：

~~~html
<img src="/generated-ticket.jpg"
     alt=x onload="eval(atob('...'))">
~~~

脚本把 document.cookie 编码后发送到选手控制的收集端。管理员 bot 预先设置可由 JavaScript 读取的 flag Cookie，再访问验证页，因此 XSS 能取得该值。实际 WP 中应使用自己的 COLLECTOR_URL，不保留比赛期间的一次性 IP。

由于被破坏的垃圾块可能偶然产生引号等字符并打断属性，官方脚本会重复申请新的随机 IV，直到生成的中间字节不干扰 payload；这是 CBC 多块构造的稳定性问题，不是另一个漏洞。

最终得到：

~~~text
shellmates{IF_Y0u_DidN't_uS3_B1t_Fl1pP1NG_atT4cK_0V3r_mUlt1_bl0CkS_th4N_PrOb4BLy_iT'S_4n_UnintEndeD_s0LUt10N}
~~~

## 方法总结

AES-CBC 只保证机密性，不自动提供完整性。看到“已知或可选明文 + 客户端可修改 IV/密文 + 解密后进入敏感解析器”时，应立即检查 bit flipping。跨多块利用必须同时考虑前驱块修改造成的上一块损坏，并让解析格式能够容忍这些垃圾数据。根本修复是使用 AEAD，或至少在解析前验证独立 MAC，同时对 HTML 属性做正确转义和加引号。
