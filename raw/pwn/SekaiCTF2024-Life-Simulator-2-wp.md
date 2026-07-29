# Life Simulator 2

## 题目简述

程序用 C++ 实现公司、项目和员工三类对象。选手可以创建或出售公司、增删项目、雇佣或解雇员工、调动员工以及推进一周。二进制开启 PIE、NX、栈 Canary 和 Full RELRO，并使用题目附带的 glibc 2.39，因此不能依赖简单的 GOT 覆写。

漏洞来自对象所有权不一致：`Company` 删除项目时会析构 `Project`，但 `Project::~Project()` 只清空自己的员工指针数组，并不删除员工；全局 `workers` 数组和每个 `Worker::project` 仍然保留原指针。只要能在公司还有员工时通过出售检查，就能制造指向已释放 `Project` 的悬空指针。

## 解题过程

### 1. 令公司预算整数回绕为零

出售公司要求满足以下任一条件：

```cpp
if (!(company->number_of_workers() == 0 ||
      company->get_company_budget() == 0)) {
    return;
}
```

每次 `elapse_week` 又按下面的公式增加无符号 64 位预算：

```cpp
profit = profit_per_week * pow(2.3L, number_of_workers);
company_budget += profit - total_salary;
```

官方利用创建资本为 `6269` 的公司和周收益为 `500` 的项目，工资全部设为零，再依次让每周员工数取：

```python
WEEK_WORKERS = [45, 44, 44, 41, 41, 38, 37, 36, 35, 34, 33,
                33, 29, 27, 25, 22, 20, 16, 14, 9, 9, 3]
```

这些离散收益在模 $2^{64}$ 的加法下恰好把公司预算推进到零。最后只保留员工 `0`，此时虽然员工数不为零，`budget == 0` 仍使 `sell_company` 放行。

出售过程会调用 `remove_projects()` 并释放项目，而员工 `0` 仍在全局数组中：

```text
workers[0] ──> Worker ──project──> 已释放的 Project
```

后续 `worker_info 0`、`move_worker 0 ...` 都会解引用这块已释放内存，形成 UAF。

### 2. 重占项目对象并泄露地址

`Project` 在 64 位 libstdc++ 下的关键布局可概括为：

```text
+0x00  std::string project_name
+0x20  uint64_t profit_per_week
+0x28  std::vector<Worker *> workers
+0x40  Company *company
```

长名字的 `std::string` 缓冲区可以重用已释放项目所在的堆块。于是把名字内容布置成一个伪造 `Project`，让其 `project_name` 的指针和长度分别指向任意地址与 `8`；调用 `worker_info 0` 时，程序会把目标地址的 8 字节当作项目名输出。

官方脚本首先通过重新分配对象并读取伪造 `workers` 向量的长度计算堆基址：

```python
heap = leaked_worker_count * 8 - 0x132f0 - 0x80
```

随后反复释放用于伪造对象的名字缓冲区，并更换 `project_name` 指针：

```python
# 读取堆中 unsorted-bin 指针，计算 libc 基址
fake_project_name(heap + 0x135d0, 8)
libc.address = leak8() - 0x203b40

# 读取 libc 的 environ，取得栈地址
fake_project_name(libc.sym.environ, 8)
stack = leak8()
```

这一步不依赖格式化字符串；泄露原语完全来自 UAF 后对 `std::string` 内部字段的伪造。

### 3. 伪造对象关系并把分配引向栈

有了堆、libc 和栈地址后，官方利用在一条超长的无效命令中放置伪造 `Company` 及其项目指针数组，再用长员工名覆盖悬空项目。最终的伪造项目满足：

- `project_name` 指向内容为 `A` 的堆地址，使 `get_project_by_name("A")` 能找到它；
- `workers` 三个指针指向受控堆区；
- `company` 指向先前布置的伪造 `Company`；
- 伪造公司的 `projects` 向量只包含目标伪造项目。

随后执行：

```text
move_worker 0 A
```

程序会在攻击者伪造的 `Project::workers` 向量上执行删除和追加。配合前面的释放顺序及 glibc 2.39 tcache 元数据布局，官方脚本把下一次大块输入缓冲区的分配位置导向 `environ` 泄露出的返回栈附近，目标取对齐后的 `stack - 0x500`。最后发送约 `0x400` 字节的 ROP 数据覆盖控制流。

ROP 链由题目所带 libc 中的 gadget 组成：先用若干 `ret` 调整栈，再执行 `pop rdi; ret`，把 libc 内的 `/bin/sh` 地址放入 `rdi`，最后调用 `system`：

```python
rop = flat(
    [libc.address + 0x2882f] * (0x300 // 8),
    libc.address + 0x10f75b,  # pop rdi ; ret
    libc.address + 0x1cb42f,  # "/bin/sh"
    libc.sym.system,
)
```

获得 shell 后读取服务端 flag 文件即可。

仓库中服务端使用的 flag 为：

```text
SEKAI{make_sure_to_pay_attention_to_your_truth_tables!_44bc24}
```

## 方法总结

本题的主线是“算术状态绕过 + C++ 对象 UAF + 伪造 STL 对象”。先利用周收益的无符号回绕，把仍有员工的公司预算变成零，从而绕过出售限制；公司释放项目后，全局员工对象留下悬空项目指针。再通过堆复用伪造 `std::string`、`std::vector` 和对象关系，依次构造堆泄露、任意 8 字节读取、libc 泄露和栈泄露，最终把受控输入写到栈上执行 ROP。

真正需要关注的不是某个固定偏移，而是对象所有权：容器里存放裸指针时，删除拥有者必须同步清除所有反向引用。否则只要攻击者能重占同尺寸堆块，正常的成员函数就会变成对伪造对象的读写接口。
