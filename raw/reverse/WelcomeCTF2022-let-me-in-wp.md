# LetMeIn

## 题目简述

附件是一个 Android APK。登录校验完全在客户端 `Login.java` 中完成：用户名先做 Base64 编码后与常量比较，密码计算无盐 MD5 后与固定摘要比较。题目不依赖 Android 权限或组件漏洞，核心是静态恢复校验逻辑。

## 解题过程

用 JADX 反编译 APK，定位按钮点击事件，可以看到：

```java
String encodedName = Base64.getEncoder()
    .encodeToString(et_username.getText().toString().getBytes());
String magic = getMd5(et_password.getText().toString());

if (encodedName.equals("R3JleWhhdEluWW91ckFyZWE=")
    && magic.equals("d23b3bc1dc24919d2439219ad6072d33")) {
    // 登录成功
}
```

Base64 常量可直接还原：

```python
import base64
print(base64.b64decode("R3JleWhhdEluWW91ckFyZWE=").decode())
```

结果为 `GreyhatInYourArea`。密码摘要是未加盐 MD5，可以用常见口令字典匹配，而不是尝试“解密”哈希：

```bash
printf '%s\n' d23b3bc1dc24919d2439219ad6072d33 > hash.txt
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 0 hash.txt --show
```

把匹配出的明文密码与用户名按官方格式组合：

```text
greyhats{GreyhatInYourArea_<password>}
```

公开仓库只保留了目标 MD5 和上述官方步骤，没有记录字典匹配出的明文；因此这里保留可复现的校验值与命令，不杜撰最终密码或 Flag。

## 方法总结

客户端登录若内置可逆用户名和快速无盐密码摘要，反编译后就失去保密性。静态分析时应沿点击处理器追踪输入的全部变换，并区分 Base64 解码与 MD5 字典匹配。即使最终验证字符串未被公开，保存精确常量、算法和组合格式仍能完整复现官方解题路径。
