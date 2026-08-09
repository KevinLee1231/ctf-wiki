# Christmas Wishes

## 题目简述

Web 页面把用户提交的 JSON 交给自制 PHP 扩展 `jsonparser.so`。解析器的 `parser_string` 存在与 Baron Samedit 类似的反斜杠/NUL 边界错误，可把短分配扩展成堆溢出；对象重复键的替换逻辑又会删除旧链表节点。覆盖旧节点的 `next`、`prev` 后，`delete_item` 中的单向摘链写入可形成任意地址写，最终把扩展自身的 `free@GOT` 改为 `system`。

虽然入口是 HTTP/PHP，决定性步骤是原生扩展的堆布局、伪造结构体和 GOT 劫持，所以归入 Pwn。

## 解题过程

### parser_string 的短分配与长复制

函数先从当前偏移向后搜索 `"` 或 NUL，并按这段表面长度分配缓冲区：

```c
while (*ptr != '"' && *ptr)
    ptr++;
int len = ptr - buf + 1;
char *out = malloc(len);
```

实际复制循环遇到反斜杠时会先移动 `buf`。若输入中出现 `\` 紧邻嵌入 NUL，长度扫描在 NUL 停止；复制阶段却由反斜杠默认分支跨过 NUL，继续复制 NUL 后、结束引号前的大段内容。于是分配长度只覆盖前缀，复制长度却覆盖完整尾部，形成可控堆溢出：

```text
"CCCCCCCCCCCCCCCCCCCCC\ NUL [padding] [fake Item_struct] "
```

### 重复键触发伪节点摘链

`Item_struct` 的尾部字段是 `name/chile/next/prev`，删除逻辑包含：

```c
if (item->prev)
    item->prev->next = item->next;
```

`next` 在结构偏移 `0x30`，所以若伪造：

```text
item->next = system_address
item->prev = free_got - 0x30
```

删除时就执行：

```text
*(free_got - 0x30 + 0x30) = system_address
```

JSON 前部用若干普通成员整理 PHP 堆，再连续放置同名键，例如先用字符串值、再用字符串值、最后改成数字。`new_Object` 会通过 `get_item_by_name` 找到旧项并调用销毁逻辑；最后一个长字符串的反斜杠/NUL 溢出覆盖邻接 `Item_struct`，重复键替换随即删除这份伪节点。

伪结构还要把会被释放的 `name` 或 `string_value` 指向可控命令字符串。GOT 改写完成后，相应的 `free(command)` 实际变成 `system(command)`。本地验证可使用无网络副作用的命令：

```text
cat /flag
```

关键构造可以抽象为：

```python
def p64_as_json_u16(value):
    raw = value.to_bytes(8, "little")
    return "".join(f"\\u{int.from_bytes(raw[i:i+2], 'big'):04x}" for i in range(0, 8, 2))

fake_item = (
    b"cat /flag\x00".ljust(0x30, b"A")
    + system_address.to_bytes(8, "little")
    + (free_got - 0x30).to_bytes(8, "little")
)
```

实际 JSON 需要按解析器的 `\uXXXX` 写入顺序编码指针，并使用与附件完全一致的 `jsonparser.so` 基址和 GOT 偏移。地址来自本地同版本容器调试，不能把官方脚本中的某次 ASLR 基址当作通用常量。

仓库镜像中的 flag 为：

```text
SCTF{YoUr_Christm@s_Wish_wi1l_c0me_true!}
```

## 方法总结

利用链由两个独立缺陷组成：反斜杠跨过 NUL，使“扫描长度”和“实际复制长度”不一致；重复键替换则把被覆盖的链表节点送入摘链代码，提供 `prev->next = next` 写原语。将 `prev` 调整到 `free@GOT-0x30`、将 `next` 设为 `system`，再让后续释放参数指向命令即可完成控制流劫持。复现时最容易出错的是 PHP 堆布局、结构偏移、JSON Unicode 写入端序和模块基址，均应在附件容器中逐项验证。
