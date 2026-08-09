# compost

## 题目简述

程序与 seedling 使用相同的平衡树和深度轮转，但比较字符串由 treenode_traverse2 生成：它按“根、左、右”的先序顺序输出，并在递归层间交替选择原字符或轮转字符。需要同时逆转遍历顺序和部分节点的字符轮转。

## 解题过程

固定串为：

~~~text
hglhhlbomrrw_nebedrcllpue
~~~

树的节点数已知，左右子树大小由中点划分唯一确定。递归消费固定串中的先序字符：当前第一个字符属于根，后面的定长区间分别属于左右子树。根层 ccase=0 使用原字符，下一层 ccase=1 使用按深度轮转的字符，之后交替。

~~~python
pos = 0

def parse(n, depth, rotated):
    global pos
    if n == 0:
        return ""
    ch = check[pos]
    pos += 1
    if rotated:
        ch = chr((ord(ch) - 32 - depth) % 95 + 32)
    left_n = n // 2
    right_n = n - left_n - 1
    left = parse(left_n, depth + 1, not rotated)
    right = parse(right_n, depth + 1, not rotated)
    return left + ch + right
~~~

最终中序拼回原输入：

~~~text
maple{hello_from_the_decompiler}
~~~

## 方法总结

逆向树遍历要区分“数据在输出中的顺序”和“原字符串中的顺序”。先序流适合递归消费，原字符串则由左子树、根、右子树重组。二进制虽已去符号，但与前置题结构相似，识别熟悉的递归形状比逐条汇编跟踪更高效。
