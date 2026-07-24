# UMDCTF 2019 - Sinking Ship

## 题目简述

附件没有扩展名，题面只称其为“奇怪的文件”。识别后可知它是一份经 gzip 压缩的 Docker 镜像导出包，flag 保存在镜像分层中。

## 解题过程

先确认格式并列出 gzip 内部的 tar：

```bash
file sinking_ship
gzip -dc sinking_ship | tar -tf -
```

输出包含 `manifest.json`、配置 JSON，以及多个 `<digest>/layer.tar`，符合 `docker save` 的旧式镜像布局。解开外层后，逐层列出文件：

```bash
mkdir image
gzip -dc sinking_ship | tar -xf - -C image

find image -name layer.tar -print0 |
  while IFS= read -r -d '' layer; do
    echo "== $layer =="
    tar -tf "$layer" | grep -E '(^|/)flag(\.txt)?$'
  done
```

其中一个 layer 保存了 `home/flag.txt`。不必启动容器，直接从该层读取：

```bash
tar -xOf image/<layer-id>/layer.tar home/flag.txt
```

文件内容是 Base64，解码后得到：

```text
UMDCTF-{th3_sh1p_sunk_but_th3_crew_surviv3d}
```

其 SHA-256 与仓库摘要一致。

## 方法总结

容器镜像是分层证据集合，删除或覆盖后的文件仍可能存在于较早的 layer 中。取证时应解析导出格式、按清单确认层顺序并直接检查各层 tar，而不是运行未知镜像。这样既能保留证据，也能避免执行镜像中的不可信代码。
