# CTF Misc - Source Backdoors and Restricted Shell Tricks

## 阅读定位

源码后门识别、受限 shell 文件读取与编辑器逃逸技巧。


## Backdoor Detection in Source Code

**模式（Rear Hatch）：** Hidden command prefix triggers `system()` call.

**Common patterns:**
- `strncmp(input, "exec:", 5)` -> runs `system(input + 5)`
- Hex-encoded comparison strings: `\x65\x78\x65\x63\x3a` = "exec:"
- Hidden conditions in maintenance/admin functions


---

## HISTFILE Trick for Restricted Shell File Reads (BCTF 2016)

Read files without cat/less/head: `HISTFILE=/flag /bin/bash && history`, or `bash -v flag.txt` (verbose mode prints lines), or `ctypes.sh` `dlcall` for direct C library calls. See bashjails.md.


---

## rvim Jail Escape via Python3 (BKP 2017)

`rvim` blocks `:!` but `:python3 import os; os.system("cmd")` executes arbitrary commands. Check `:version` for `+python3`/`+lua`/`+ruby`. See interactive-containers-jails-and-solvers.md.
