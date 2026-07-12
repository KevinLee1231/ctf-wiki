# 1！5！

## 题目简述

题目给出一份流量包和一份 Linux 内存镜像。题面提示“服务器一直发送奇怪消息，流量里的消息完全加密，L1near 做了服务器镜像”，并明确说明真正 flag 在流量里，内存里的 flag 是假的。解题需要把网络取证和内存取证结合起来：先从 TCP/WebSocket 流量中看到自定义加密数据，再从内存镜像中恢复服务端运行环境、`eval.js` 加密逻辑和 `SSLKEYLOGFILE` 内容，分别解出 WebSocket 前半段与 HTTP/3 QUIC 后半段，最后拼接 flag。

## 解题过程

1. 打开流量，可以发现它由 QUIC 和 TCP 两部分组成。TCP 部分存在 WebSocket 通信，并且每段消息都是加密后的字符串；QUIC 部分后续需要 TLS key log 才能解开。

3.将其全部提取作为备用，手工或者脚本都可以，可以看得出来是一些加密内容，但是不知道加密方式，先留着，看看内存

4.分析内存，发现是lime镜像，意味着是linux系统镜像

```
strings memory.mem |grep 'Linux version'
```

获得关键信息，Linux version 4.19.0-21-amd64，且发现是debian系统。对照 Linux 发行版内核支持表可知，`4.19.0-21-amd64` 对应 Debian 10 系列内核，因此后续制作 Volatility Linux profile 时按 Debian 10 环境准备符号。

5.下载其iso并安装后，制作其对应的符号表镜像 以方便后续取证

```
git clone https://github.com/volatilityfoundation/volatility.git
cd volatility
pip2 install pycrypto
pip2 install distorm3
cd tools/linux
make
cd ../../
zip volatility/plugins/overlays/linux/Debian10.zip tools/linux/module.dwarf /boot/System.map-4.19.0-21-amd64
```

最后使用 `python2 vol.py --info | grep debian` 即可发现符号表制作成功。

6.进行内存取证，查看一下历史命令

```
python2 vol.py -f ../memory.mem --profile=Linuxdebian10x64 linux_bash
```

7. 根据历史记录，可以发现服务器运行了另一个 HTTP/3 服务端，这与流量里的 QUIC 吻合；历史命令还记录了 `SSLKEYLOGFILE` 的路径，位置在桌面目录，并且最后出现了 `eval.js`，说明内存里可能同时有 QUIC 解密材料和 WebSocket 加密脚本。

8. 查看进程可以确认目标运行了 `apache2` 服务，再次遇见 `eval.js`，说明该脚本比较重要。尝试使用 `linux_find_file` 查找目标文件。

再次遇见了eval.js说明比较重要，尝试使用linux_find_file指令进行查找

9. 出于 Volatility 对该镜像的限制，并不能通过 `linux_find_file` 稳定找到目标文件的缓存地址。

由于内存镜像本质的原理，而且我们已经掌握了关键信息，我们通过strings来快速过滤我们的关键内容

10. 通过 `strings memory.mem | grep eval` 快速定位相关信息。可以看到大量 `eval.js` 相关内容，并在附近发现一段以 `eval` 形式加载的 JavaScript 代码。

结合前后数据的位置，可以确认该段js就是eval.js的内容，将其赋值出来进行反混淆。

11.通过对其反混淆，可以获得如下代码

```js
function randomString(e) {
    e = e || 32;
    var t = "ABCDEFGHJKMNPQRSTWXYZabcdefhijkmnprstwxyz2345678",
        a = t.length,
        n = "";
    for (i = 0; i < e; i++) n += t.charAt(Math.floor(Math.random() * a));
    return n
}

function encrypto(a, b, c) {
    if (typeof a !== 'string' || typeof b !== 'number' || typeof c !== 'number') {
        return
    }
    let resultList = [];
    c = c <= 25 ? c : c % 25;
    for (let i = 0; i < a.length; i++) {
        let charCode = a.charCodeAt(i);
        charCode = (charCode * 1) ^ b;
        charCode = charCode.toString(c);
        resultList.push(charCode)
    }
    let splitStr = String.fromCharCode(c + 97);
    let resultStr = resultList.join(splitStr);
    return resultStr
}
var b1 = new Encode();
var ws = new WebSocket("ws://localhost:2303/flag");
ws.onopen = function(a) {
    console.log("Connection open ...");
    ws.send("flag")
};
ws.onmessage = function(a) {
    var b = randomString(5);
    var n = a.data;
    var res = n.padEnd(9, b);
    var s1 = encrypto(res, 15, 25);
    var f1 = b1.encode(s1);
    ws.send(f1);
    console.log('Connection Send:' + f1)
};
ws.onclose = function(a) {
    console.log("Connection closed.")
};

function Encode() {
    _keyStr = "/128GhIoPQROSTeUbADfgHijKLM+n0pFWXY456xyzB7=39VaqrstJklmNuZvwcdEC";
    this.encode = function(a) {
        var b = "";
        var c, chr2, chr3, enc1, enc2, enc3, enc4;
        var i = 0;
        a = _utf8_encode(a);
        while (i < a.length) {
            c = a.charCodeAt(i++);
            chr2 = a.charCodeAt(i++);
            chr3 = a.charCodeAt(i++);
            enc1 = c >> 2;
            enc2 = ((c & 3) << 4) | (chr2 >> 4);
            enc3 = ((chr2 & 15) << 2) | (chr3 >> 6);
            enc4 = chr3 & 63;
            if (isNaN(chr2)) {
                enc3 = enc4 = 64
            } else if (isNaN(chr3)) {
                enc4 = 64
            }
            b = b + _keyStr.charAt(enc1) + _keyStr.charAt(enc2) + _keyStr.charAt(enc3) + _keyStr.charAt(enc4)
        }
        return b
    }
    _utf8_encode = function(a) {
        a = a.replace(/\r\n/g, "\n");
        var b = "";
        for (var n = 0; n < a.length; n++) {
            var c = a.charCodeAt(n);
            if (c < 128) {
                b += String.fromCharCode(c)
            } else if ((c > 127) && (c < 2048)) {
                b += String.fromCharCode((c >> 6) | 192);
                b += String.fromCharCode((c & 63) | 128)
            } else {
                b += String.fromCharCode((c >> 12) | 224);
                b += String.fromCharCode(((c >> 6) & 63) | 128);
                b += String.fromCharCode((c & 63) | 128)
            }
        }
        return b
    }
}
```

12.经过分析，可以发现其流程为：websocket连接服务端，向其发送flag字段，然后服务端向html发送明文flag，通过加密再次发送出去

加密流程：首先随机生成字符串补在flag字段后面，然后进行了异或的加密，最后进行了换表的base64操作。

至此我们可以同样写一串js代码来解密其字段

13.解密代码

```js
<script>
var str1 ="待解密的字符串"
function Base64() {
    var _keyStr = "/128GhIoPQROSTeUbADfgHijKLM+n0pFWXY456xyzB7=39VaqrstJklmNuZvwcdEC";

    this.decode = function(input) {
        var output = "";
        var chr1, chr2, chr3;
        var enc1, enc2, enc3, enc4;
        var i = 0;
        input = input.replace(/[^A-Za-z0-9\+\/\=]/g, "");
        while (i < input.length) {
            enc1 = _keyStr.indexOf(input.charAt(i++));
            enc2 = _keyStr.indexOf(input.charAt(i++));
            enc3 = _keyStr.indexOf(input.charAt(i++));
            enc4 = _keyStr.indexOf(input.charAt(i++));
            chr1 = (enc1 << 2) | (enc2 >> 4);
            chr2 = ((enc2 & 15) << 4) | (enc3 >> 2);
            chr3 = ((enc3 & 3) << 6) | enc4;
            output = output + String.fromCharCode(chr1);
            if (enc3 != 64) {
                output = output + String.fromCharCode(chr2);
            }
            if (enc4 != 64) {
                output = output + String.fromCharCode(chr3);
            }
        }
        output = _utf8_decode(output);
        return output;
    }
 
    var _utf8_decode = function (utftext) {
        var string = "";
        var i = 0;
        var c = 0;
        var c1 = 0;
        var c2 = 0;
        while (i < utftext.length) {    
            c = utftext.charCodeAt(i);
            if (c < 128) {
                string += String.fromCharCode(c);
                i++;
            } else if ((c > 191) && (c < 224)) {
                c1 = utftext.charCodeAt(i + 1);
                string += String.fromCharCode(((c & 31) << 6) | (c1 & 63));
                i += 2;
            } else {
                c1 = utftext.charCodeAt(i + 1);
                c2 = utftext.charCodeAt(i + 2);
                string += String.fromCharCode(((c & 15) << 12) | ((c1 & 63) << 6) | (c2 & 63));
                i += 3;
            }
        }
        return string;
    }
}

function decrypto( str, xor, hex ) { 
  if ( typeof str !== 'string' || typeof xor !== 'number' || typeof hex !== 'number') {
    return;
  }
  let strCharList = [];
  let resultList = []; 
  hex = hex <= 25 ? hex : hex % 25;
  let splitStr = String.fromCharCode(hex + 97);
  strCharList = str.split(splitStr);

  for ( let i=0; i<strCharList.length; i++ ) {

    let charCode = parseInt(strCharList[i], hex);

    charCode = (charCode * 1) ^ xor;
    let strChar = String.fromCharCode(charCode);
    resultList.push(strChar);
  }
  let resultStr = resultList.join('');
  return resultStr;
}
 
var base = new Base64()
b64 = base.decode(str1)
console.log(b64)
s1 = decrypto(b64,15,25)
console.log(s1[0])


</script>
```

14.成功解密前半段，获得前半段flag:WMCTF{LOL_StR1ngs_1s_F@ke_BUT

15.流量还有quic流量需要解密，结合前面得知的sslkeylog.txt，我们同样可以通过strings来快速锁定。NSS/SSLKEYLOGFILE 的作用是记录 TLS 会话密钥，Wireshark 可以读取该文件解密 TLS/QUIC 流量；TLS 1.3/QUIC 常见字段包括 `CLIENT_HANDSHAKE_TRAFFIC_SECRET`、`SERVER_HANDSHAKE_TRAFFIC_SECRET`、`CLIENT_TRAFFIC_SECRET_0`、`SERVER_TRAFFIC_SECRET_0`，每行格式是“标签、client random、secret”。

那么只需要一次通过strings来过滤如下

```
CLIENT_HANDSHAKE_TRAFFIC_SECRET
SERVER_HANDSHAKE_TRAFFIC_SECRET
CLIENT_TRAFFIC_SECRET_0
SERVER_TRAFFIC_SECRET_0
```

四个关键段即可

16.最终拼接的sslkeylog的内容为

```
CLIENT_HANDSHAKE_TRAFFIC_SECRET 1002eec63c7da0d66827ebc83af50e00550704d76420b1d039f9ef2222641dd2 48f1197d22ef93778c14f15ddbbf9a53df20cf74c9c68b9f3073fa9f405da995
SERVER_HANDSHAKE_TRAFFIC_SECRET 1002eec63c7da0d66827ebc83af50e00550704d76420b1d039f9ef2222641dd2 38b4671e9ded337c7066e3830563f4519f3bf4effb13d046c2e62847329f0787
CLIENT_TRAFFIC_SECRET_0 1002eec63c7da0d66827ebc83af50e00550704d76420b1d039f9ef2222641dd2 457d3990a971aad9a308ea0af62db5745d99a75e0c484487289f9e760b33a43f
SERVER_TRAFFIC_SECRET_0 1002eec63c7da0d66827ebc83af50e00550704d76420b1d039f9ef2222641dd2 dc730355e51308929f66eabb06458080459810bdd6b27de884a1c1fdc5385b1e
```

17. 最后在 Wireshark 的 TLS 设置中填入拼接好的 key log 文件，即可解密 HTTP/3 流量。

18. 解密后的 HTTP/3 流量中可以看到后半段 flag：`_HTTP3_1s_C000L}`。

19.最终flag为：WMCTF{LOL_StR1ngs_1s_F@ke_BUT_HTTP3_1s_C000L}

## 方法总结

本题核心是“流量里有密文，内存里有解密材料”。TCP/WebSocket 部分要从内存中恢复 `eval.js`，得到“随机补齐、异或 15、25 进制字符串、换表 base64”的加密流程；QUIC/HTTP3 部分要从内存中恢复 TLS 1.3 key log，再交给 Wireshark 解密。

识别信号是：pcap 同时包含 WebSocket 和 QUIC；内存镜像为 LiME Linux；bash 历史中出现 HTTP/3 服务、`SSLKEYLOGFILE` 和 `eval.js`；题面明确说内存里的 flag 是假的，说明内存应作为钥匙而不是最终答案来源。

复现时先制作匹配 Debian 10 `4.19.0-21-amd64` 的 Volatility profile，再用 `linux_bash` 和 `strings` 定位服务脚本与 key log。WebSocket 解密后得到前半段 `WMCTF{LOL_StR1ngs_1s_F@ke_BUT`，HTTP/3 解密得到后半段，最终 flag 为 `WMCTF{LOL_StR1ngs_1s_F@ke_BUT_HTTP3_1s_C000L}`。

参考资料要点：Linux 内核支持表用于把 `4.19.0-21-amd64` 对应到 Debian 10；NSS Key Log Format 文档说明 SSLKEYLOGFILE 可供 Wireshark 解密 TLS 流量，TLS 1.3 需要提取握手和应用流量 secret。原文分别见 https://files.trendmicro.com/documentation/guides/deep_security/Kernel%20Support/12.0/Deep_Security_12_0_kernels_EN.html 和 https://nss-crypto.org/reference/security/nss/legacy/key_log_format/index.html。
