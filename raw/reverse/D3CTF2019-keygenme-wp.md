# KeyGenMe

## 题目简述

题目使用 MIRACL 实现 DSA 授权校验。调试函数生成题目时会用同一个随机数 `k` 分别签名消息 `flag` 和 `d3ctf`，其中 `flag` 的签名片段作为目标 flag 输出，`d3ctf` 的签名 `<r,s>` 和 `k` 被写入 `signed.out`，私钥文件随后删除。选手需要根据已知消息、签名和泄露的 `k` 还原 DSA 私钥 `x`，再重新对 `flag` 签名，通过游戏授权校验拿到题目 flag。

题目资料地址：https://github.com/yype/D3CTF_Rev/tree/master/pushBox

题目主体其实是一个 `pushBox` 游戏，`debug` 功能故意留在二进制中给玩家分析。程序启动时会检查 `signed.out` 是否为输入名称的合法签名；如果已验证的名称为 `flag`，游戏会打印 `<r,s>` 的一部分作为题目 flag。私钥恢复公式正是 $x \equiv r^{-1}(ks-H(m)) \pmod q$，与正文中的 MIRACL/DSA 推导一致。

## 解题过程

题目用开源库 [MIRACL](https://github.com/miracl/MIRACL) 为游戏实现了一个有弱点的 DSA 签名授权校验， `debug` 函数被用来生成题目，生成题目的过程中会输出对 `flag` 的签名（即为该题 flag），同时对 `d3ctf` 进行签名，保留的文件只有对 `d3ctf` 的签名，选手需要还原对 `flag` 的签名。

函数 `debug` 的步骤：

- Generate a new random common.dss which contains p, q and g
- Generate a new random public.dss and private.dss which contains public and private keys
- Generate a random `k`
- Use the generated `k` to sign message `flag`, print part of the <r,s> pair as the flag
- Use the same `k` to sign message `d3ctf`, write <r,s> to the signature file `signed.out`
- Write `k` to the signature file `signed.out`, remove `private.dss`

载入游戏前会校验授权（ `signed.out` 是否为输入名称的签名），如果是则启动游戏，启动后判断输入名称是否为 `flag` ，正确则输出题目的 flag。

根据泄露的 `k` 和已知的签名还原 `x` ，再对 `flag` 进行签名即可：

$x \equiv r^{-1}(ks - H(m)) \pmod{q}$

#### Solution

```c
#include <stdio.h>
#include <memory.h>
#include <miracl/miracl.h>
#include <stdlib.h>
#include <string.h>

void hashing(char* msg, int msg_len, big hash)
{
    char h[20];
    int i,ch;
    sha sh;
    shs_init(&sh);
    for(i = 0;i<msg_len;i++){
        shs_process(&sh,msg[i]);
    }
    shs_hash(&sh,h);
    bytes_to_big(20,h,hash);
}

int main(){
    FILE *fp;
    big p,q,g,x,y,r,s,k,hash,tmp0,tmp1;
    char msg[] = "d3ctf";
    char flag[] = "flag";
    long seed;
    int bits;
    miracl *mip;

    fp=fopen("common.dss","rt");
    if (fp==NULL) {
        printf("file common.dss does not existn");
        return 0;
    }
    fscanf(fp,"%dn",&bits);
    mip=mirsys(bits/4,16);
    p=mirvar(0);
    q=mirvar(0);
    g=mirvar(0);
    x=mirvar(0);
    y=mirvar(0);
    r=mirvar(0);
    s=mirvar(0);
    k=mirvar(65535);
    hash=mirvar(0);
    tmp0=mirvar(0);
    tmp1=mirvar(0);

    innum(p,fp);
    innum(q,fp);
    innum(g,fp);
    fclose(fp);

    hashing(msg, strlen(msg), hash);
    fp=fopen("signed.out","rt");
    if (fp==NULL) {
        printf("file signed.out does not existn");
        return 0;
    }

    innum(r, fp);
    innum(s, fp);
    innum(k,fp);

    xgcd(k,q,k,k,k);

    fclose(fp);
    tmp1 = r;
    xgcd(r,q,r,r,r); // 1/r mod q
    multiply(k,s,tmp0);
    subtract(tmp0,hash,tmp0);
    multiply(r,tmp0,x);
    divide(x,q,q); // mod q

    hashing(flag, strlen(flag), hash);
    xgcd(r,q,r,r,r); // inverse the inverse of r

    xgcd(k,q,k,k,k); // 1/k mod q
    mad(x,r,hash,q,q,s);
    mad(s,k,k,q,q,s);

    FILE* fd = fopen("signed.out","wt");

    otnum(r,fd);
    otnum(s,fd);
    printf("Done.n");

    return 0;
}
```

## 方法总结

- 核心技巧：DSA 一旦泄露或复用 nonce `k`，可由 `x = r^{-1}(ks - H(m)) mod q` 恢复私钥，再伪造任意消息签名。
- 识别信号：签名文件同时包含 `<r,s>` 和 `k`，或多个消息使用相同 `r`，应立即检查 DSA/ECDSA nonce 复用问题。
- 复用要点：实现时要确认 MIRACL 的哈希、整数输入输出格式和模逆方向；恢复私钥后不要只改 `s`，需要按目标消息 `flag` 重新计算完整签名并写回授权文件。

