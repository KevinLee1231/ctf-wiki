# miniLCTF 2024 Snooker King Writeup

## 题目简述

题目提供一个 Cocos Creator 网页台球游戏，表面目标是通过击球达到指定分数。真正的敏感数据和胜利阈值被直接序列化进前端场景资源，因此无需实际完成游戏。

## 解题过程

浏览器开发者工具的 Network/Sources 面板中搜索 `score`、`target` 或 `miniLCTF`，可以定位场景文件：

```text
assets/main/import/0e/0e3a51e5c.json
```

Cocos Creator 会把组件字段按类型表和数据索引压入同一个 JSON。格式虽然不适合人工阅读，但字符串和数字常量仍是明文。该文件末段包含：

```text
target = 1145
flag   = miniLCTF{U_ju3T_Hlt_1145_sN0Ok3r_Balls?!}
```

也可直接在下载后的前端目录搜索：

```powershell
rg -n "miniLCTF|1145|target" "web-desktop/assets"
```

最终 flag 为：

```text
miniLCTF{U_ju3T_Hlt_1145_sN0Ok3r_Balls?!}
```

原题解中的开发者工具截图只展示这些文本，没有额外视觉关系，故归档时直接转写字段，不保留截图。

## 方法总结

纯前端游戏不能把秘密放在客户端资源中。Cocos、Unity WebGL 等引擎即使采用序列化索引，也只是在改变存储结构，并未提供保密性。先全文搜索构建产物中的 flag 格式、阈值和组件字段，通常比逆向完整游戏逻辑更高效。
