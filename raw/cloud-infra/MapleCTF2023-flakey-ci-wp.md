# flakey-ci

## 题目简述

服务接收一个 HTTPS Git 仓库地址，并执行 `nix build --accept-flake-config` 构建其中的 flake。构建进程看似处于 Nix 沙箱内，但服务账号被配置在 `nix.settings.trusted-users` 中，攻击者提供的 flake 因而可以让守护进程接受高权限的 `nixConfig` 选项。

决定性配置是 `post-build-hook`：它不是普通 derivation 内的脚本，而是在构建结束后由 Nix 守护进程调用，可以访问沙箱外的主机文件。

## 解题过程

恶意仓库准备两个阶段。第一阶段定义一个 derivation，把读取 `/flag` 并向外发送内容的 hook 脚本写入 Nix store。这样脚本获得一个稳定、只含哈希和名称的绝对 store 路径。

第二阶段在 `flake.nix` 中加入：

```nix
nixConfig = {
  post-build-hook = "/nix/store/<hash>-exfil-hook";
};
```

并提供一个能正常完成的小型构建目标。服务使用 `--accept-flake-config`，而调用用户又是 trusted user，因此这个客户端提交的全局配置不会被忽略。derivation 构建结束后，daemon 在沙箱外执行 hook。题目虚拟机通过 QEMU fw_cfg 把真正 flag 暴露并链接为 `/flag`，hook 可以直接读取它，再使用题目允许的出站方式回传。

触发一次 CI 构建并查看接收端即可得到：

```text
maple{trust3d_us3rs_ar3_r00t_4c7u411y}
```

## 方法总结

Nix 构建沙箱只约束 derivation，并不自动约束 daemon 的管理钩子。`trusted-users` 实际上允许用户影响一批近似 root 权限的配置，应视为高权限边界；`--accept-flake-config` 又把不受信任仓库中的声明提升为客户端配置。安全 CI 不应让外部 flake控制全局 hook、substituter 或签名信任设置，并应把拉取代码与有特权的 Nix daemon 隔离。
