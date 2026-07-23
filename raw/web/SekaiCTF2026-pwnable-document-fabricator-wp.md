# pwnable document fabricator

## 题目简述

应用接收任意上传文件，并调用 `@pdftron/pdfnet-node` 11.13.0 将其转换为 PDF：

```javascript
const pdfdoc = await PDFNet.PDFDoc.create()
await PDFNet.Convert.printerSetMode(
  PDFNet.Convert.PrinterMode.e_prefer_builtin_converter
)
await PDFNet.Convert.toPdf(pdfdoc, req.file.path)
await pdfdoc.save(outputPath, PDFNet.SDFDoc.SaveOptions.e_linearized)
```

这里明确优先使用 PDFNet 自带转换器，而不是宿主打印服务。题面说明这是尚未修复的 0day，比赛统计为 0 解；仓库也没有 `solution/`、PoC、崩溃样本或漏洞说明。因此这篇只能给出可由源码证明的攻击面和复现边界，不能写成已经存在完整利用。

## 解题过程

从源码能够确定的攻击面如下：

1. `/convert` 无认证地接收文件。
2. Multer 没有配置上传大小上限；原文件名的扩展名只保留字母，但没有格式白名单。
3. 文件以随机 UUID 名保存到 `/tmp`，内容原样进入 PDFNet 内置的多格式转换路径。
4. PDFNet 是随 Node 包加载的原生组件；解析异常不在 JavaScript 沙箱内。
5. 转换结果仍写入同一 `/tmp`，随后由 Express 作为 PDF 下载。
6. 容器以 `node` 用户运行，根文件系统只读、capabilities 全部丢弃、禁止提权且 egress 关闭；只有 `/tmp` 可写，`/readflag` 为 root 所有但模式是 `0111`。

因此至少需要两个原语：

1. 构造一个能进入 PDFNet 内置转换器的恶意文档，在解析或转换过程中控制进程并执行 `/readflag`；
2. 在没有出站网络的条件下，把输出带回本次 `/convert` 响应，例如影响最终下载的 PDF 或建立其他同站 in-band 通道。

第二点不能被省略：只有命令执行而把 flag 打到容器 stdout，并不等于参赛者能读取它。仓库证据不足以确定具体 bug 是内存破坏、命令注入、任意文件操作还是多条原语组合，也不能确定应由哪种输入格式触发。

本题不能从现有材料准确重建可复现 payload。为了避免把猜测写成事实，本文不伪造 CVE、偏移、堆布局或利用代码。仓库的 `kona.yaml` 与构建时排除的 `readflag.c` 保存了比赛 flag：

```text
SEKAI{th3r3_w3r3_l1k3_4t_l34s7_4_d0z3n_p4wbl3ms_h3r3_th4t_1_r3p0rt3d_l1k3_cm0n_wh4t_4r3_w3_d01ng_br0}
```

这只能证明预期成功结果，不能替代缺失的 0day 利用链。

## 方法总结

本题的公开入口、数据流和题面定位均是 Web，放在 Web 类合理；真正缺失的决定性原语则位于原生文档转换组件。对未公开且零解的 0day，准确的 WP 应明确区分“源码可证攻击面”“利用必须具备的能力”和“尚无证据的具体漏洞”，尤其不能把仓库中可见的最终 flag 误写成通过某个杜撰 PoC 得到的结果。
