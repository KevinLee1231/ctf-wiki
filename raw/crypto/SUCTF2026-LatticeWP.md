# SUCTF2026-Lattice

## 题目简述
题目是逆向出的高位泄露 LFSR/线性递推问题。程序从 `data` 读取模数、24 个反馈系数和 24 个初始状态，hint 返回后续状态的高位；要从多次高位泄露恢复最初状态或提交值。附件解压后主要是挑战数据/密文类材料，价值在于作为输入数据，不应把全文写进题目描述；正文应保留逆向恢复出的递推关系、泄露位数和格建模。

## 解题过程
题目分析

这题表面上是一个菜单交互题：

- 选项 1 ：提交答案拿 flag

- 选项 2 ：获取 hint

- 选项 3 ：退出

真正的难点在于我们拿不到内部状态，只能不断拿 hint，然后反推出应该提交的那个答案。

对二进制 chall 做逆向后，可以恢复出核心逻辑：

1. 程序会从 `./data` 中读取三部分内容：模数 $m$、24 个反馈系数 $c_0,\ldots,c_{23}$、24 个初始状态 $a_0,\ldots,a_{23}$。

2. 它维护的是一个 24 阶 Fibonacci Z/(m)-LFSR

3. 每次请求 hint 时，先计算下一项

4. 递推关系为：

$$
a_{i+24} \equiv \sum_{j=0}^{23} c_j a_{i+j} \pmod m
$$

5. 然后返回这一个新状态的高位：

$$
\text{hint}_i = a_{i+24} \gg 20
$$

6. 提交答案时，程序要求的并不是当前状态，而是最初那 24 个初始状态之和：

$$
\text{answer} = \sum_{i=0}^{23} a_i \pmod m
$$

所以题目本质就是：

已知一个 24 阶 Fibonacci Z/(m) -LFSR 的连续高位截断输出，恢复未知的 m 、反馈系数和初始状
态，再计算最初 24 项的和。

参数识别

由逆向结果可以直接确定：

- 阶数 n = 24

- 模数位长约为 60

- 每个 hint 泄露高 40 位

- 低 20 位未知

记 $a_i = 2^\beta y_i + z_i$，其中 $y_i$ 是泄露的高位，$z_i$ 是未知低位。

则这里有：

- alpha = 40
- beta = 20
- k = alpha + beta = 60

这正好对应论文 2025-2323.pdf 第 3.2 节讨论的场景：

模数未知，但模数接近 2 的幂。

为什么不能直接爆破模数

一开始最自然的想法是枚举 m 在 2^60 附近的候选值，然后对每个候选跑已知模数攻击。

这个思路在本地小范围样本上可以过，但远端不稳定。原因是论文只保证：

$$
2^k - m < 2^\beta \quad \text{or} \quad m - 2^{k-1} < 2^\beta
$$

也就是说，模数不一定只落在 2^60 附近，也可能落在 2^59 附近。 因此简单扫一个很窄的
2^60 \pm 2^{10} 区间是不够的，必须按照论文的 unknown modulus 方法做。

论文对应的攻击思路

2. 用高位截断值构造 L_{alpha,y}

论文第 3.2 节给出了 unknown modulus 场景下的格，可理解为由连续高位窗口 $Y_i$ 和 $2^\alpha I_t$ 组成的块矩阵：

$$
L_{\alpha,y} =
\begin{pmatrix}
2^\alpha I_t & Y_0 & 1 \\
               & Y_1 & 1 \\
               & \vdots & \vdots \\
               & Y_{r-1} & 1
\end{pmatrix}
$$

其中：

$$
Y_i = (y_i, y_{i+1}, \ldots, y_{i+t-1})
$$

如果约减后得到短向量，对应的系数

$$
\eta = (\eta_0, \ldots, \eta_{r-1})
$$

就会满足一组 annihilating relation，从而构成一个整系数多项式

$$
F(x)=\eta_{r-1}x^{r-1}+\cdots+\eta_1x+\eta_0
$$

它实际上是序列在 Z/(m) 上的 annihilating polynomial。

3. 用 resultant 的 gcd 恢复模数

论文里的关键结论是：如果拿到足够多的 annihilating polynomials，那么任意两两 resultant 都会
被 m^n 整除。

因此可以：

1. 从 $L_{\alpha,y}$ 的约减基里取出多组多项式。

2. 计算若干个两两 resultant。

3. 对这些 resultant 取 gcd。

这样就能得到一个被 m^24 整除的大整数，从中筛回真正的模数。

由于本题满足 “模数接近 2 的幂” 的条件，所以只需要在两个窗口内筛：

- $2^{60} - 2^{20}$ 到 $2^{60}$
- $2^{59}$ 到 $2^{59} + 2^{20}$

这一步直接把 unknown modulus 转成了 known modulus。

4. 已知模数后恢复反馈多项式

模数一旦恢复，就切换到论文第 3.1 节的 known modulus 场景。

构造格 $L_{m,y}$，核心是把模数 $m$、截断低位界 $2^\beta$ 和连续高位窗口 $Y_i$ 放进同一个嵌入矩阵。对它做 BKZ，可以得到多组 annihilating polynomials。

把这些多项式在 mod $m$ 意义下转成首一多项式，再不断做 gcd，就能恢复出真正的 24 阶特征多项式：

$$
f(x)=x^{24}-c_{23}x^{23}-\cdots-c_1x-c_0
$$

5. 恢复低 20 位并还原初始状态

已知反馈多项式后，初始状态的未知部分只剩每项低 20 位。 这一步对应论文里提到的 Kannan
embedding / SIS 转 SVP 思路。

做法是：

1. 用恢复出来的反馈关系构造 companion matrix

2. 建立低位未知量的嵌入格

3. 再做一次 BKZ

4. 直接恢复最前面 24 项的低位

从而得到“当前观测窗口”的完整状态。

6. 从观测窗口倒推回最初 24 项

远端返回的 hint 对应的是“下一项”的高位，所以恢复出来的完整状态其实是一个右移后的窗口。 我
们要的答案却是最原始的那 24 项之和。

因此还需要把递推反过来做 24 步。

因为 Fibonacci 形式满足

$$
a_{i+24} \equiv c_{23}a_{i+23}+\cdots+c_1a_{i+1}+c_0a_i \pmod m
$$

只要 c_0 在模 m 下可逆，就能反推出前一项：

$$
a_i \equiv c_0^{-1}\left(a_{i+24}-\sum_{j=1}^{23} c_j a_{i+j}\right) \pmod m
$$

这样倒推 24 次，就回到了最初读入 data 的那组初始状态，最后求和即可。

参数选择

论文给出的理论下界可以概括为：known modulus 场景需要泄露位数、序列阶数和格参数满足短向量可分离条件；unknown modulus 场景还要额外保证模数接近 2 的幂。本题的参数满足这两个条件。

代入本题参数：

- n = 24
- log m ≈ 60
- alpha = 40
- beta = 20

出题人 blog 中按论文条件写成近似不等式：

$$
\frac{1}{r}+\frac{1}{t}\le \frac{\alpha}{n\log(m)}
$$

代入本题参数后，取 $r=t>72$ 即可满足理论条件；出题人脚本使用了 83。当前实现为了减少 BKZ 和 resultant 阶段的不稳定性，参数取得更保守：

- r = 88
- t = 88
- hints = 200

这个配置在本地与远端都稳定通过。

实现细节

solve.py

solve.py 只负责交互：

1. 连接本地进程或远端 socket

2. 对 10001 先发送一个 \r

3. 连续请求 200 个 hints

4. 把 hints 写入临时文件

5. 调用 recover_candidate

6. 提交最终答案

这里有一个远端细节很坑：

- 10001 端口不是连上就立即出菜单

- 必须先发一个回车

- 首屏通常还要等十几秒

如果脚本没有这个唤醒动作，就会看起来像“卡死”。

recover_candidate.cpp

helper 的流程是：

1. 用 L_{alpha,y} 从高位序列中提取整系数 annihilating polynomials

2. 用多组 resultant 的 gcd 恢复模数候选

3. 对候选模数走 known modulus 恢复

4. 在 mod m 下求反馈多项式 gcd

5. 用嵌入格恢复状态低位

6. 反推回最初状态并求和

实现时做了几个工程化处理：

- 对 L_{alpha,y} 和 L_{m,y} 都直接使用 BKZ_FP

- 从 reduced basis 中提取多行，而不是只赌第一行

- 对整系数多项式先做 PrimitivePart
- resultant 只要累计若干个非零值即可，不必全算完

- 模数候选只在论文允许的两个窗口内筛，避免无意义爆炸

这题的关键不在“继续多拿一些 hint”，而在于要先正确识别模型：

- 它不是普通线性递推

- 而是高位截断的 Fibonacci Z/(m) -LFSR

- 并且模数未知但接近 2 的幂

只要识别到这点，整题就和论文第 3.2 节完全对上：

1. 高位格 L_{alpha,y} 找 annihilating polynomials

2. resultant gcd 找模数

3. known modulus 格恢复反馈多项式

4. 嵌入格恢复低位状态

5. 逆递推拿回最初 24 项

这也是为什么最后的核心并不是 binary exploitation，而是一个比较完整的 lattice + truncated LFSR
参数恢复题。

recover_candidate.cpp

```c
#include <NTL/LLL.h>
#include <NTL/ZZ.h>
#include <NTL/ZZX.h>
#include <NTL/ZZ_pX.h>
#include <NTL/mat_ZZ.h>
#include <NTL/vec_ZZ.h>

#include <algorithm>
#include <fstream>
#include <iostream>
#include <list>
#include <set>
#include <sstream>
#include <string>
#include <vector>

NTL_CLIENT

namespace {

constexpr int kOrder = 24;
constexpr int kAlpha = 40;
constexpr int kBeta = 20;
constexpr int kBitLength = kAlpha + kBeta;
constexpr int kKnownSearchR = 88;
constexpr int kKnownSearchT = 88;
constexpr int kUnknownSearchR = 88;

constexpr int kUnknownSearchT = 88;
constexpr int kRecoverDigits = 44;
constexpr int kResultantPolyLimit = 12;
constexpr int kRequiredNonZeroResultants = 6;

ZZ positive_mod(const ZZ& value, const ZZ& modulus) {
ZZ result = value % modulus;
if (result < 0) {
result += modulus;
}
return result;
}

vec_ZZ read_hints(const std::string& path) {
std::ifstream fin(path);
if (!fin) {
throw std::runtime_error("failed to open hints file");
}

std::vector<ZZ> values;
long long hint = 0;
while (fin >> hint) {
values.emplace_back(hint);
}

vec_ZZ hints;
hints.SetLength(values.size());
for (long i = 0; i < static_cast<long>(values.size()); ++i) {
hints[i] = values[i];
}
return hints;
}

mat_ZZ search_linear_relations_high_m(const ZZ& modulus, const vec_ZZ& hints,
int beta, int r, int t) {
mat_ZZ lattice, candidates;
lattice.SetDims(r + t, r + t);
candidates.SetDims(r + t, r);

clear(lattice);
for (int i = 0; i < t; ++i) {
lattice[i][i] = modulus;
}
for (int i = 0; i < r; ++i) {
const int row = t + i;
lattice[row][row] = power2_ZZ(beta);
for (int j = 0; j < t; ++j) {

lattice[row][j] = hints[i + j] * power2_ZZ(beta);
}
}

BKZ_FP(lattice, 0.99, 20);

for (int i = 0; i < r + t; ++i) {
for (int j = 0; j < r; ++j) {
candidates[i][j] = lattice[i][j + t] / power2_ZZ(beta);
}
}

return candidates;
}

mat_ZZ search_linear_relations_power2(const vec_ZZ& hints, int alpha, int r,
int t) {
mat_ZZ lattice, candidates;
lattice.SetDims(r + t, r + t);
candidates.SetDims(r + t, r);

clear(lattice);
for (int i = 0; i < t; ++i) {
lattice[i][i] = power2_ZZ(alpha);
}
for (int i = 0; i < r; ++i) {
const int row = t + i;
lattice[row][row] = 1;
for (int j = 0; j < t; ++j) {
lattice[row][j] = hints[i + j];
}
}

BKZ_FP(lattice, 0.99, 20);

for (int i = 0; i < r + t; ++i) {
for (int j = 0; j < r; ++j) {
candidates[i][j] = lattice[i][j + t];
}
}

return candidates;
}

ZZX row_to_integer_polynomial(const mat_ZZ& candidates, long row) {
ZZX poly;
for (long col = 0; col < candidates.NumCols(); ++col) {

if (!IsZero(candidates[row][col])) {
SetCoeff(poly, col, candidates[row][col]);
}
}
return poly;
}

std::string serialize_poly(const ZZX& poly) {
std::ostringstream oss;
oss << deg(poly) << ':';
for (long i = 0; i <= deg(poly); ++i) {
oss << coeff(poly, i) << ',';
}
return oss.str();
}

std::vector<ZZX> extract_integer_polynomials(const mat_ZZ& candidates, int
min_degree, int limit) {
std::vector<ZZX> polys;
std::set<std::string> seen;

for (long row = 0; row < candidates.NumRows(); ++row) {
ZZX poly = row_to_integer_polynomial(candidates, row);
if (deg(poly) < min_degree) {
continue;
}
poly = PrimitivePart(poly);
if (deg(poly) < min_degree) {
continue;
}

const std::string key = serialize_poly(poly);
if (!seen.insert(key).second) {
continue;
}

polys.push_back(poly);
if (static_cast<int>(polys.size()) >= limit) {
break;
}
}

return polys;
}

ZZ_pX integer_to_monic_mod_poly(const ZZX& poly) {
ZZ_pX mod_poly;

for (long i = 0; i <= deg(poly); ++i) {
if (!IsZero(coeff(poly, i))) {
SetCoeff(mod_poly, i, conv<ZZ_p>(coeff(poly, i)));
}
}

if (deg(mod_poly) < 0) {
return mod_poly;
}

const ZZ_p lead = LeadCoeff(mod_poly);
if (IsZero(lead)) {
clear(mod_poly);
return mod_poly;
}

mod_poly *= inv(lead);
return mod_poly;
}

ZZ_pX recover_coefficients(const mat_ZZ& candidates, const ZZ& modulus, int n)
{
ZZ_p::init(modulus);

std::vector<ZZ_pX> monic_polys;
monic_polys.reserve(candidates.NumRows());

for (long row = 0; row < candidates.NumRows(); ++row) {
ZZX poly = row_to_integer_polynomial(candidates, row);
if (deg(poly) < n) {
continue;
}
poly = PrimitivePart(poly);
if (deg(poly) < n) {
continue;
}

ZZ_pX mod_poly = integer_to_monic_mod_poly(poly);
if (deg(mod_poly) >= n) {
monic_polys.push_back(mod_poly);
}
}

if (monic_polys.size() < 2) {
return ZZ_pX();
}

for (long i = 0; i < static_cast<long>(monic_polys.size()); ++i) {
for (long j = i + 1; j < static_cast<long>(monic_polys.size()); ++j) {
ZZ_pX gcd_poly = GCD(monic_polys[i], monic_polys[j]);
if (deg(gcd_poly) < n) {
continue;
}

for (long k = 0; k < static_cast<long>(monic_polys.size()) &&
deg(gcd_poly) > n; ++k) {
if (k == i || k == j) {
continue;
}
ZZ_pX next = GCD(gcd_poly, monic_polys[k]);
if (deg(next) >= n) {
gcd_poly = next;
}
}

if (deg(gcd_poly) == n) {
return gcd_poly;
}
}
}

return ZZ_pX();
}

vec_ZZ recover_initial_state(const vec_ZZ& hints, const ZZ_pX& poly, const ZZ&
modulus, int n, int digits, int beta) {
vec_ZZ state, low_bits;
mat_ZZ companion, companion_power, lattice;

state.SetLength(n);
low_bits.SetLength(n);
companion.SetDims(n, n);
companion_power.SetDims(n, n);
lattice.SetDims(digits + 1, digits + 1);

clear(companion);
clear(lattice);
companion_power = ident_mat_ZZ(n);

companion[0][n - 1] = positive_mod(-rep(poly[0]), modulus);
for (int i = 1; i < n; ++i) {
companion[i][i - 1] = 1;
companion[i][n - 1] = positive_mod(-rep(poly[i]), modulus);
}

for (int i = 1; i < n; ++i) {
companion_power = companion_power * companion;
for (int row = 0; row < n; ++row) {
for (int col = 0; col < n; ++col) {
companion_power[row][col] = positive_mod(companion_power[row]
[col], modulus);
}
}
}

for (int i = n; i < digits; ++i) {
companion_power = companion_power * companion;
for (int row = 0; row < n; ++row) {
for (int col = 0; col < n; ++col) {
companion_power[row][col] = positive_mod(companion_power[row]
[col], modulus);
}
}

ZZ acc(0);
for (int j = 0; j < n; ++j) {
lattice[j + 1][i + 1] = companion_power[j][0];
acc += companion_power[j][0] * hints[j];
}
lattice[0][i + 1] = positive_mod(power2_ZZ(beta) * (hints[i] - acc),
modulus) + power2_ZZ(beta - 1);
}

lattice[0][0] = power2_ZZ(beta - 1);
for (int i = 1; i <= n; ++i) {
lattice[0][i] = power2_ZZ(beta - 1);
lattice[i][i] = 1;
}
for (int i = n + 1; i <= digits; ++i) {
lattice[i][i] = modulus;
}

BKZ_FP(lattice, 0.99, 20);

if (lattice[0][0] == -power2_ZZ(beta - 1)) {
for (int i = 0; i < n; ++i) {
low_bits[i] = lattice[0][i + 1] + power2_ZZ(beta - 1);
state[i] = hints[i] * power2_ZZ(beta) + low_bits[i];
}
} else if (lattice[0][0] == power2_ZZ(beta - 1)) {
for (int i = 0; i < n; ++i) {

low_bits[i] = power2_ZZ(beta - 1) - lattice[0][i + 1];
state[i] = hints[i] * power2_ZZ(beta) + low_bits[i];
}
} else {
clear(state);
}

return state;
}

std::vector<ZZ> recurrence_coefficients(const ZZ_pX& poly, const ZZ& modulus) {
std::vector<ZZ> coeffs(kOrder);
for (int i = 0; i < kOrder; ++i) {
coeffs[i] = positive_mod(-rep(poly[i]), modulus);
}
return coeffs;
}

bool validate_solution(const vec_ZZ& hints, const ZZ& modulus, const ZZ_pX&
poly, const vec_ZZ& shifted_state, int beta) {
if (shifted_state.length() != kOrder) {
return false;
}

const auto coeffs = recurrence_coefficients(poly, modulus);
const ZZ scale = power2_ZZ(beta);
std::list<ZZ> window;

for (int i = 0; i < kOrder; ++i) {
if (shifted_state[i] < 0 || shifted_state[i] >= modulus) {
return false;
}
if (shifted_state[i] / scale != hints[i]) {
return false;
}
window.push_back(shifted_state[i]);
}

for (long index = kOrder; index < hints.length(); ++index) {
ZZ next(0);
int pos = 0;
for (const auto& value : window) {
next += coeffs[pos] * value;
++pos;
}
next = positive_mod(next, modulus);
if (next / scale != hints[index]) {

return false;
}
window.pop_front();
window.push_back(next);
}

return true;
}

ZZ recover_original_sum(const vec_ZZ& shifted_state, const ZZ_pX& poly, const
ZZ& modulus) {
const auto coeffs = recurrence_coefficients(poly, modulus);
if (GCD(coeffs[0], modulus) != 1) {
throw std::runtime_error("c0 is not invertible modulo m");
}

const ZZ c0_inv = InvMod(coeffs[0], modulus);
std::list<ZZ> window;
for (int i = 0; i < kOrder; ++i) {
window.push_back(shifted_state[i]);
}

for (int step = 0; step < kOrder; ++step) {
std::vector<ZZ> current(window.begin(), window.end());
ZZ prev = current.back();
for (int j = 1; j < kOrder; ++j) {
prev -= coeffs[j] * current[j - 1];
}
prev = positive_mod(prev * c0_inv, modulus);
window.pop_back();
window.push_front(prev);
}

ZZ answer(0);
for (const auto& value : window) {
answer = positive_mod(answer + value, modulus);
}
return answer;
}

bool solve_with_modulus(const vec_ZZ& hints, const ZZ& modulus, ZZ& answer) {
if (!ProbPrime(modulus)) {
return false;
}
if (hints.length() < kKnownSearchR + kKnownSearchT - 1 || hints.length() <
kRecoverDigits) {
return false;

}

const mat_ZZ candidates = search_linear_relations_high_m(modulus, hints,
kBeta, kKnownSearchR, kKnownSearchT);
const ZZ_pX poly = recover_coefficients(candidates, modulus, kOrder);
if (deg(poly) != kOrder) {
return false;
}

const vec_ZZ shifted_state = recover_initial_state(hints, poly, modulus,
kOrder, kRecoverDigits, kBeta);
if (!validate_solution(hints, modulus, poly, shifted_state, kBeta)) {
return false;
}

answer = recover_original_sum(shifted_state, poly, modulus);
return true;
}

ZZ gcd_of_resultants(const std::vector<ZZX>& polys) {
ZZ gcd_resultant(0);
int nonzero = 0;

for (long i = 0; i < static_cast<long>(polys.size()); ++i) {
for (long j = i + 1; j < static_cast<long>(polys.size()); ++j) {
ZZ resultant_value;
resultant(resultant_value, polys[i], polys[j]);
if (IsZero(resultant_value)) {
continue;
}
resultant_value = abs(resultant_value);
if (IsZero(gcd_resultant)) {
gcd_resultant = resultant_value;
} else {
gcd_resultant = GCD(gcd_resultant, resultant_value);
}
++nonzero;
if (nonzero >= kRequiredNonZeroResultants &&
NumBits(gcd_resultant) >= 60 * kOrder) {
return gcd_resultant;
}
}
}

return gcd_resultant;
}

void append_divisors_in_band(std::vector<ZZ>& moduli, const ZZ& value, unsigned
long long start, unsigned long long end) {
for (unsigned long long candidate = start; candidate <= end; ++candidate) {
const ZZ candidate_zz(candidate);
if (candidate_zz <= 1) {
continue;
}
if (value % candidate_zz != 0) {
continue;
}
if (value % power(candidate_zz, kOrder) != 0) {
continue;
}
moduli.push_back(candidate_zz);
}
}

std::vector<ZZ> recover_modulus_candidates(const std::vector<ZZX>& polys) {
const ZZ gcd_resultant = gcd_of_resultants(polys);
if (IsZero(gcd_resultant)) {
return {};
}

std::vector<ZZ> moduli;
const unsigned long long delta = 1ULL << kBeta;
append_divisors_in_band(moduli, gcd_resultant, (1ULL << kBitLength) -
delta, 1ULL << kBitLength);
append_divisors_in_band(moduli, gcd_resultant, 1ULL << (kBitLength - 1),
(1ULL << (kBitLength - 1)) + delta - 1);

std::sort(moduli.begin(), moduli.end(), [](const ZZ& lhs, const ZZ& rhs) {
return lhs < rhs; });
moduli.erase(std::unique(moduli.begin(), moduli.end()), moduli.end());
return moduli;
}

bool solve_unknown_modulus(const vec_ZZ& hints, ZZ& answer) {
if (hints.length() < kUnknownSearchR + kUnknownSearchT - 1 ||
hints.length() < kRecoverDigits) {
return false;
}

const mat_ZZ unknown_candidates = search_linear_relations_power2(hints,
kAlpha, kUnknownSearchR, kUnknownSearchT);
const auto polys = extract_integer_polynomials(unknown_candidates, kOrder,
kResultantPolyLimit);
if (polys.size() < 2) {

return false;
}

const auto moduli = recover_modulus_candidates(polys);
for (const auto& modulus : moduli) {
if (solve_with_modulus(hints, modulus, answer)) {
return true;
}
}

return false;
}

} // namespace

int main(int argc, char** argv) {
try {
if (argc == 3) {
const ZZ modulus(INIT_VAL, argv[1]);
const vec_ZZ hints = read_hints(argv[2]);
ZZ answer;
if (!solve_with_modulus(hints, modulus, answer)) {
return 1;
}
std::cout << answer << std::endl;
return 0;
}

if (argc == 2) {
const vec_ZZ hints = read_hints(argv[1]);
ZZ answer;
if (!solve_unknown_modulus(hints, answer)) {
return 1;
}
std::cout << answer << std::endl;
return 0;
}

std::cerr << "usage: recover_candidate [modulus] <hints_file>\n";
return 2;
} catch (const std::exception& ex) {
std::cerr << ex.what() << std::endl;
return 4;
}
}
```

solve.py

```python
#!/usr/bin/env python3

import argparse
import socket
import subprocess
import sys
import tempfile
from pathlib import Path
from typing import Protocol

ROOT = Path(__file__).resolve().parent
HELPER_SRC = ROOT / "recover_candidate.cpp"
HELPER_BIN = ROOT / "recover_candidate"
CHALL_BIN = ROOT / "chall"
HINT_COUNT = 200

class ChallengeIO(Protocol):
def read_char(self) -> str: ...
def write(self, data: str) -> None: ...
def close(self) -> None: ...

class LocalChallenge:
def __init__(self) -> None:
self.proc = subprocess.Popen(
[str(CHALL_BIN)],
stdin=subprocess.PIPE,
stdout=subprocess.PIPE,
stderr=subprocess.STDOUT,
text=True,
bufsize=0,
)

def read_char(self) -> str:
assert self.proc.stdout is not None
return self.proc.stdout.read(1)

def write(self, data: str) -> None:
assert self.proc.stdin is not None
self.proc.stdin.write(data)
self.proc.stdin.flush()

def close(self) -> None:
if self.proc.poll() is not None:
return
try:
self.write("3\n")
self.proc.wait(timeout=1)
except Exception:
self.proc.kill()

class RemoteChallenge:
def __init__(self, host: str, port: int) -> None:
self.sock = socket.create_connection((host, port))

def read_char(self) -> str:
data = self.sock.recv(1)
return data.decode() if data else ""

def write(self, data: str) -> None:
self.sock.sendall(data.encode())

def close(self) -> None:
try:
self.write("3\n")
except OSError:
pass
self.sock.close()

def build_helper() -> None:
needs_build = not HELPER_BIN.exists() or HELPER_BIN.stat().st_mtime <
HELPER_SRC.stat().st_mtime
if not needs_build:
return
cmd = ["g++", "-O2", str(HELPER_SRC), "-lntl", "-lgmp", "-o",
str(HELPER_BIN)]
subprocess.run(cmd, check=True)

def read_until(io: ChallengeIO, token: str) -> str:
chunks = []
while True:
char = io.read_char()
if char == "":
raise RuntimeError("challenge closed unexpectedly")
chunks.append(char)
if "".join(chunks).endswith(token):

return "".join(chunks)

def read_until_any(io: ChallengeIO, tokens: list[str]) -> str:
chunks = []
while True:
char = io.read_char()
if char == "":
return "".join(chunks)
chunks.append(char)
current = "".join(chunks)
if any(current.endswith(token) for token in tokens):
return current

def get_hints(io: ChallengeIO, count: int) -> list[int]:
hints: list[int] = []
# The remote service on port 10001 waits for an initial carriage return
# before it prints the first menu.
io.write("\r")
read_until(io, ">>> ")
for _ in range(count):
io.write("2\n")
output = read_until(io, ">>> ")
marker = "Here is your hint: "
start = output.find(marker)
if start == -1:
raise RuntimeError(f"failed to parse hint from: {output!r}")
start += len(marker)
end = output.find("\n", start)
hints.append(int(output[start:end]))
return hints

def recover_answer(hints: list[int]) -> int:
with tempfile.NamedTemporaryFile("w", delete=False) as tmp:
hint_file = Path(tmp.name)
tmp.write("\n".join(map(str, hints)))

try:
proc = subprocess.run(
[str(HELPER_BIN), str(hint_file)],
stdout=subprocess.PIPE,
stderr=subprocess.PIPE,
text=True,
check=False,
)

if proc.returncode != 0:
detail = proc.stderr.strip() or proc.stdout.strip() or "helper
```

failed"

```python
raise RuntimeError(detail)
return int(proc.stdout.strip().splitlines()[-1])
finally:
try:
hint_file.unlink(missing_ok=True)
except OSError:
pass

def submit_answer(io: ChallengeIO, answer: int) -> str:
io.write("1\n")
read_until(io, "Please enter your answer: ")
io.write(f"{answer}\n")
tail = read_until_any(io, [">>> "])
return tail

def open_challenge(host: str | None, port: int | None) -> ChallengeIO:
if host is None and port is None:
return LocalChallenge()
if host is None or port is None:
raise ValueError("--host and --port must be provided together")
return RemoteChallenge(host, port)

def main() -> int:
parser = argparse.ArgumentParser(description="Solve the challenge through
interaction only.")
parser.add_argument("--hints", type=int, default=HINT_COUNT, help="number
of hints to collect")
parser.add_argument("--host", help="remote host")
parser.add_argument("--port", type=int, help="remote port")
args = parser.parse_args()

build_helper()
io = open_challenge(args.host, args.port)
try:
hints = get_hints(io, args.hints)
answer = recover_answer(hints)
result = submit_answer(io, answer)
sys.stdout.write(result)
finally:
io.close()
return 0

if __name__ == "__main__":
raise SystemExit(main())

# python solve.py --host <target> --port 10001
```

## 方法总结
- 核心技巧：高位泄露线性递推的格恢复
- 识别信号：能反复拿到 LFSR/线性递推状态高位，低位未知但有界。
- 复用要点：逆向出递推式后，用高位 hint 建立近似线性关系，把低位误差放进格中恢复状态。
