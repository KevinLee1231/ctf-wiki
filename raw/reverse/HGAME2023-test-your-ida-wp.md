# test_your_ida

## 题目简述

这是一道 IDA 入门题。官方 PDF 只写明“IDA 打开即可看到 flag”；参赛者的[复盘文章](https://blog.vvbbnn00.cn/archives/hgame2023week1-bu-fen-ti-jie)补充了反编译结果：程序读取至多 10 个字符，与常量 `r3ver5e` 比较，匹配时输出内置的 flag 字符串。

## 解题过程

将附件载入 IDA，定位 `main`。为便于理解，可以把反编译结果整理成下面的等价伪代码：

```c
int main(void) {
    char input[24];

    scanf("%10s", input);
    if (strcmp(input, "r3ver5e") == 0) {
        puts("your flag:hgame{te5t_y0ur_IDA}");
    }
    return 0;
}
```

这道题不需要恢复加密算法。既可以在伪代码中沿 `strcmp` 的成功分支查看输出，也可以打开 Strings 窗口搜索 `your flag`，再跟随交叉引用回到 `main`。输入：

```text
r3ver5e
```

程序输出：

```text
hgame{te5t_y0ur_IDA}
```

## 方法总结

- 核心技巧：从 `main`、字符串常量和交叉引用快速定位输入校验与成功分支。
- 识别信号：flag 以明文常量直接编译进程序，没有经过编码或运行时拼接。
- 复用要点：入门逆向题先检查 Strings 和导入函数，再沿 `strcmp`、`puts` 等调用回溯，通常比逐条阅读汇编更高效。
