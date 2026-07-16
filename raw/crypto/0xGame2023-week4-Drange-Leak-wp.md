# Drange Leak

## 题目简述

题目生成 2048 位 RSA 模数 $n=pq$，但私钥指数不是随机选取，而是

$$
d=M d_1+d_0,
$$

其中已泄露 $M$，且 $d_1<2^{60}$、$d_0<2^{70}$。目标是把这个近似线性结构代入 RSA 私钥关系，构造多元小根问题，恢复 $p+q$ 后分解 $n$。

## 解题过程

由 $ed\equiv1\pmod{\varphi(n)}$，存在整数 $k$ 使：

$$
e(Md_1+d_0)-1=k\varphi(n).
$$

又因为

$$
\varphi(n)=(p-1)(q-1)=n-(p+q-1),
$$

令 $z=p+q-1$，则：

$$
eMd_1+ed_0-1-k(n-z)=0.
$$

对模 $eM$ 取同余，含 $d_1$ 的第一项消失：

$$
ed_0-1-kn+kz\equiv0\pmod{eM}.
$$

于是可在 $\mathbb Z/(eM)\mathbb Z$ 上构造三元多项式：

$$
f(x,k,z)=ex-kn+kz-1,
$$

其目标根为 $(d_0,k,p+q-1)$。已知 $d_0<2^{70}$；由 $k=(ed-1)/\varphi(n)$ 且 $e<\varphi(n)$，可取 $k<d<2^{1015}$；两个 1024 位素数之和严格小于 $2^{1025}$。原脚本把 $z$ 的缩放值写成 $2^{1024}$，这是偏紧的 LLL 启发式参数，不是严格上界；以下使用数学上正确的 $2^{1025}$，若格规约不稳定，可围绕该值微调缩放和 `m`。

附件中的多元小根实现与完整收尾如下：

```sage
import itertools
from Crypto.Util.number import long_to_bytes


def small_roots(f, bounds, m=1, degree=None):
    if degree is None:
        degree = f.degree()

    ring = f.base_ring()
    modulus = ring.cardinality()
    f /= f.coefficients().pop(0)
    f = f.change_ring(ZZ)

    polynomials = Sequence([], f.parent())
    for i in range(m + 1):
        base = modulus ** (m - i) * f ** i
        for shifts in itertools.product(
            range(degree), repeat=f.nvariables()
        ):
            polynomials.append(
                base * prod(
                    power(variable, shift)
                    for variable, shift in zip(f.variables(), shifts)
                )
            )

    lattice, monomials = polynomials.coefficient_matrix()
    monomials = vector(monomials)
    factors = [monomial(*bounds) for monomial in monomials]

    for column, factor in enumerate(factors):
        lattice.rescale_col(column, factor)
    lattice = lattice.dense_matrix().LLL()
    lattice = lattice.change_ring(QQ)
    for column, factor in enumerate(factors):
        lattice.rescale_col(column, 1 / factor)

    equations = Sequence([], f.parent().change_ring(QQ))
    for equation in filter(None, lattice * monomials):
        equations.append(equation)
        ideal = equations.ideal()
        if ideal.dimension() == -1:
            equations.pop()
        elif ideal.dimension() == 0:
            return [
                tuple(ring(solution[var]) for var in f.variables())
                for solution in ideal.variety(ring=ZZ)
            ]
    return []


# n、e、c、M 使用题目输出中的四个整数。
n = 20890649807098098590988367504589884104169882461137822700915421138825243082401073285651688396365119177048314378342335630003758801918471770067256781032441408755600222443136442802834673033726750262792591713729454359321085776245901507024843351032181392621160709321235730377105858928038429561563451212831555362084799868396816620900530821649927143675042508754145300235707164480595867159183020730488244523890377494200551982732673420463610420046405496222143863293721127847196315699011480407859245602878759192763358027712666490436877309958694930300881154144262012786388678170041827603485103596258722151867033618346180314221757
e = 18495624691004329345494739768139119654869294781001439503228375675656780205533832088551925603457913375965236666248560110824522816405784593622489392063569693980307711273262046178522155150057918004670062638133229511441378857067441808814663979656329118576174389773223672078570346056569568769586136333878585184495900769610485682523713035338815180355226296627023856218662677851691200400870086661825318662718172322697239597148304400050201201957491047654347222946693457784950694119128957010938708457194638164370689969395914866589468077447411160531995194740413950928085824985317114393591961698215667749937880023984967171867149
c = 7268748311489430996649583334296342239120976535969890151640528281264037345919563247744198340847622671332165540273927079037288463501586895675652397791211130033797562320858177249657627485568147343368981852295435358970875375601525013288259717232106253656041724174637307915021524904526849025976062174351360431089505898256673035060020871892556020429754849084448428394307414301376699983203262072041951835713075509402291301281337658567437075609144913905526625759374465018684092236818174282777215336979886495053619105951835282087487201593981164477120073864259644978940192351781270609702595767362731320959397657161384681459323
M = 136607909840146555806361156873618892240715868885574369629522914036807393164542930308166609104735002945881388216362007941213298888307579692272865700211608126496105057113506756857793463197250909161173116422723246662094695586716106972298428164926993995948528941241037242367190042120886133717

R.<x, k, z> = PolynomialRing(Zmod(e * M))
f = e * x - k * n + k * z - 1
roots = small_roots(f, (2^100, 2^1024, 2^1025), m=3, degree=3)

for d0, multiplier, z_value in roots:
    z_value = ZZ(z_value)
    p_plus_q = z_value + 1
    delta = p_plus_q^2 - 4 * n
    if delta >= 0 and delta.is_square():
        root_delta = delta.sqrt()
        p = (p_plus_q + root_delta) // 2
        q = (p_plus_q - root_delta) // 2
        if p * q == n:
            phi = (p - 1) * (q - 1)
            d = inverse_mod(e, phi)
            print(long_to_bytes(pow(c, d, n)))
            break
```

恢复 $z$ 后，令 $S=z+1=p+q$，则 $p$、$q$ 是方程 $t^2-St+n=0$ 的两个整数根。分解模数并正常 RSA 解密，得到：

```text
0xGame{a9e1f260f845be84f56ff06b165deb80}
```

## 方法总结

泄露量 $M$ 的价值在于模 $eM$ 后可以消去未知的 $d_1$，只留下三个有界未知量 $d_0$、$k$ 和 $p+q-1$。多元 Coppersmith/LLL 负责恢复小根；拿到 $p+q$ 后应立即用二次方程验证并分解 $n$。参数 `bounds` 是格缩放的一部分，必须区分严格数学上界与为了规约效果采用的经验值。
