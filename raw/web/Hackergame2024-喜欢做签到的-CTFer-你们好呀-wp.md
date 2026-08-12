# 喜欢做签到的 CTFer 你们好呀

## 题目简述

题目提示两个 flag 藏在中国科学技术大学校内 CTF 战队的招新主页中。该主页模拟 Linux 终端，但所有命令和输出都由浏览器端实现，因此既可以按终端语义寻找环境变量和隐藏文件，也可以直接审计打包后的 JavaScript；两处 flag 均只是 Base64 编码，并非加密。

## 解题过程

### 模拟终端路径

进入主页后输入 `help` 查看支持的命令。`env` 会输出模拟环境变量，其中 `FLAG` 的值就是第一个 flag：

```text
ctfer@ustc-nebula:$ ~ env
PWD=/root/Nebula-Homepage
ARCH=loong-arch
NAME=Nebula-Dedicated-High-Performance-Workstation
OS=NixOS❄️
FLAG=flag{actually_theres_another_flag_here_trY_to_f1nD_1t_y0urself___join_us_ustc_nebula}
REQUIREMENTS=1. you must come from USTC; 2. you must be interested in security!
```

接着使用 `ls -al`，而不是默认会隐藏点文件的 `ls`，可以看到 `.flag`。读取它得到第二个 flag：

```text
ctfer@ustc-nebula:$ ~ cat .flag
flag{0k_175_a_h1dd3n_s3c3rt_f14g___please_join_us_ustc_nebula_anD_two_maJor_requirements_aRe_shown_somewhere_else}
```

### 前端源码路径

在开发者工具的 Sources 中搜索 `atob(`、`FLAG=` 或较长的 Base64 字符串，可以找到两段类似代码：

```javascript
return atob("...");
return "PWD=/root/Nebula-Homepage\n...\nFLAG=" + atob("...");
```

把字符串复制出来解码即可：

```python
import base64

encoded = b"..."  # 替换为源码中的字符串
print(base64.b64decode(encoded).decode())
```

源码路径与模拟终端路径得到的内容一致，说明网页只是把预设文件系统和命令输出包装成终端界面。

## 方法总结

- 核心技巧：一是按 Unix 语义检查环境变量和点文件，二是审计客户端脚本并解码 `atob` 使用的 Base64 常量。
- 识别信号：网页终端无需真实后端、命令集合固定、静态脚本中出现 `atob` 和完整输出模板。
- 复用要点：前端资源对用户完全可见；Base64 仅改变表示形式。遇到模拟终端时既要尝试 `env`、`ls -al` 等常见信息发现命令，也要检查其实现源码。
