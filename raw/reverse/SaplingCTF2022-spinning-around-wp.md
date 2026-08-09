# Spinning Around

## 题目简述

程序先按固定顺序插入字符，构造一棵二叉搜索树。随后逐字符处理 flag：找到对应节点后，通过旋转把该节点移到根，并记录整棵树的序列化形状。附件给出每一步期望树状态，需要从这些状态反推出字符序列。

## 解题过程

关键是 move-to-root 操作是确定性的。给定当前树和候选字符，搜索路径决定执行左旋还是右旋，直到候选节点成为根。因此可以逐位枚举允许字符，而不必一次搜索整个 flag：

~~~python
alphabet = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_{}!"
tree = build_initial_tree(alphabet)
answer = ""

for expected in expected_states:
    for candidate in alphabet:
        trial = clone(tree)
        move_to_root(trial, candidate)
        if serialize(trial) == expected:
            answer += candidate
            tree = trial
            break
    else:
        raise ValueError("no candidate")
~~~

每找到一个字符，必须保留旋转后的树作为下一轮状态；若每轮都从初始树开始，会在很早的位置失配。恢复出的完整字符串为：

~~~text
maple{r0t4ting_unb4lanc3d_tr3es_4r3_v3ry_very_fun}
~~~

把结果重新送入原变换，逐轮序列化结果与附件完全一致。

## 方法总结

状态机逆向适合做“逐步候选 + 正向模拟”：当每一步输出能唯一约束当前输入时，搜索复杂度从指数级降为字符集大小乘长度。树旋转会改变后续状态，复制、序列化和更新顺序必须与原程序一致。最可靠的验证是用恢复输入完整重放所有状态。
