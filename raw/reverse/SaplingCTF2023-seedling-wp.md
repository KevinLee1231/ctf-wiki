# seedling

## 题目简述

源码把输入字符串递归构造成平衡二叉树。每个节点同时保存原字符和按树深度轮转后的字符，再按中序遍历输出轮转字符，与固定 check_str 比较。中序遍历仍保持原字符串顺序，因此只需计算每个位置对应的树深度并逆向字符轮转。

## 解题过程

源码中的字符变换为：

$$
R(c,n)=(c-32+n)\bmod95+32
$$

逆变换就是把 n 改成 -n。递归构树每次选择区间中点，因而可直接在字符串索引上复刻深度：

~~~python
check = "iqexvngduyvzfywdbshdizrhvnssvdjtre~szudzng|msjdtqffwzujcnpdkmlivf"
out = [""] * len(check)

def recover(lo, hi, level):
    if lo >= hi:
        return
    mid = lo + (hi - lo) // 2
    out[mid] = chr((ord(check[mid]) - 32 - level) % 95 + 32)
    recover(lo, mid, level + 1)
    recover(mid + 1, hi, level + 1)

recover(0, len(check), 0)
print("".join(out))
~~~

得到 flag 主体并按程序格式包装：

~~~text
maple{classic_structs_and_functions_for_your_viewing_pleasure_in_ghidra}
~~~

## 方法总结

看到树结构时不必真的分配节点；只要还原递归区间与遍历顺序，就能在数组上直接求解。这里中序遍历保持字符相对顺序，唯一未知量是每个节点深度。轮转字符集包含 ASCII 32 至 126，共 95 个字符，取模不能误写成 26。
