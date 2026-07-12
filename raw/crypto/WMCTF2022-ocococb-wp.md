# ocococb

## 题目简述

题目是 “AES.OCB again?” 形式的交互服务，提供自选 nonce/message 加密、提交 nonce/cipher/tag 解密，以及最终提交 secret 换 flag 的功能。漏洞组合有两点：一是 OCB 类模式中 nonce 可复用，导致相同 nonce 下的 offset/keystream 关系可被多次查询复用；二是解密路径的 unpad/错误处理可作为 oracle，允许逐块爆破 secret。脚本先用前两次加密恢复 $E(\text{nonce})$ 或等价 offset，再构造后续查询获得所需的 $E(\text{message})$、PMAC 中间量和合法 tag，最后借助 unpad 行为恢复 secret。

## 解题过程

由于 nonce 可复用，利用前两次加密获取 $E(\text{nonce})$，然后用第三次加密获取到所有我们需要的 $E(\text{message})$。接着利用 unpad 的漏洞对 secret 进行爆破，最后提交即可获得 flag。exp如下：

```python
from base64 import *
from Crypto.Util.number import *
from gmpy2 import *
from pwn import *
import math

table = '1234567890qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM'
def passpow():
	rev = r.recvuntil("sha256(XXXX+")
	suffix = r.recv(16).decode()
	r.recvuntil(" == ")
	res = r.recv(64).decode()
	def f(x):
		hashresult = hashlib.sha256((x+suffix).encode()).hexdigest()
		if hashresult == res:
			return 1
		else:
			return 0
	prefix = util.iters.mbruteforce(f,table,4,'upto')
	r.recvuntil("XXXX: ")
	r.sendline(str(prefix))

def times2(input_data,blocksize = 16):
    assert len(input_data) == blocksize
    output =  bytearray(blocksize)
    carry = input_data[0] >> 7
    for i in range(len(input_data) - 1):
        output[i] = ((input_data[i] << 1) | (input_data[i + 1] >> 7)) % 256
    output[-1] = ((input_data[-1] << 1) ^ (carry * 0x87)) % 256
    assert len(output) == blocksize
    return output

def times3(input_data):
    assert len(input_data) == 16
    output = times2(input_data)
    output = xor_block(output, input_data)
    assert len(output) == 16
    return output

def back_times2(output_data,blocksize = 16):
    assert len(output_data) == blocksize
    input_data =  bytearray(blocksize)
    carry = output_data[-1] & 1
    for i in range(len(output_data) - 1,0,-1):
        input_data[i] = (output_data[i] >> 1) | ((output_data[i-1] % 2) << 7)
    input_data[0] = (carry << 7) | (output_data[0] >> 1)
    if(carry):
        input_data[-1] = ((output_data[-1] ^ (carry * 0x87)) >> 1) | ((output_data[-2] % 2) << 7)
    assert len(input_data) == blocksize
    return input_data

def xor_block(input1, input2):
    assert len(input1) == len(input2)
    output = bytearray()
    for i in range(len(input1)):
        output.append(input1[i] ^ input2[i])
    return output

def hex_to_bytes(input):
    return bytearray(long_to_bytes(int(input,16)))

def my_pmac(header, offset, blocksize = 16):
    assert len(header)
    header = bytearray(header)
    m = int(max(1, math.ceil(len(header) / float(blocksize))))
    # offset = Arbitrary_encrypt(bytearray([0] * blocksize))
    offset = times3(offset)
    offset = times3(offset)
    checksum = bytearray(blocksize)
    offset = times2(offset)
    H_m = header[((m - 1) * blocksize):]
    assert len(H_m) <= blocksize
    if len(H_m) == blocksize:
        offset = times3(offset)
        checksum = xor_block(checksum, H_m)
    else:
        H_m.append(int('10000000', 2))
        while len(H_m) < blocksize:
            H_m.append(0)
        assert len(H_m) == blocksize
        checksum = xor_block(checksum, H_m)
        offset = times3(offset)
        offset = times3(offset)
    final_xor = xor_block(offset, checksum)
    # auth = Arbitrary_encrypt(final_xor)
    # return auth
    return final_xor

r=remote("1.13.189.168","32086")

def talk1(nonce, message):
    r.recvuntil("[-] ")
    r.sendline("1")
    r.recvuntil("[-] ")
    r.sendline(b64encode(nonce))
    r.recvuntil("[-] ")
    r.sendline(b64encode(message))
    r.recvuntil("ciphertext: ")
    ciphertext = b64decode(r.recvline(False).strip())
    r.recvuntil("tag: ")
    tag = b64decode(r.recvline(False).strip())
    return ciphertext, tag

def talk2(nonce, cipher, tag):
    r.recvuntil("[-] ")
    r.sendline("2")
    r.recvuntil("[-] ")
    r.sendline(b64encode(nonce))
    r.recvuntil("[-] ")
    r.sendline(b64encode(cipher))
    r.recvuntil("[-] ")
    r.sendline(b64encode(tag))
    r.recvuntil("plaintext: ")
    message = b64decode(r.recvline(False).strip())
    return message

context(log_level='debug')
passpow()

finalnonce = b'\x00'*16
m1 = b"\x00"*15 + b"\x80"
m2 = b"\x10"*16
cipher1, finaltag = talk1(finalnonce,m1)
cipher2, _ = talk1(finalnonce,b'')
cipher2 = xor_block(m2,cipher2)
E1 = back_times2(xor_block(cipher1[:16],cipher2))
assert back_times2(times2(E1))
# E1 = E(finalnonce)
print(E1)

def get_enc(message, offest):
    cnt = 0
    finalmessage = b''
    for i in message:
        cnt += 1
        E = offest
        for _ in range(cnt):
            E = times2(E)
        # print(cnt)
        finalmessage += xor_block(i,E)
    # print(finalmessage)
    data, tag = talk1(finalnonce,finalmessage)
    cipher = []
    cnt = 0
    for i in range(0,len(data),16):
        cnt += 1
        E = offest
        for _ in range(cnt):
            E = times2(E)
        # print(cnt)
        cipher.append(xor_block(data[i:i+16],E))
    return cipher[:-1]

xor1 = my_pmac(b'from baby',E1)
xor2 = my_pmac(b"from admin",E1)

xor_all = xor_block(m1,m2)
message = [xor1,xor2,xor_block(times2(times2(times2(E1))),m1)]
for i in range(16,32):
    print(b'\x10'*15+long_to_bytes(i))
    message.append(xor_block(times2(E1),(xor_block(xor_all,bytearray(b'\x10'*15+long_to_bytes(i))))))
# message = [xor1,xor2]
source_cipher = (get_enc(message,E1))
print(source_cipher)
finaltag = xor_block(finaltag,xor_block(source_cipher[0],source_cipher[1]))
source_cipher = source_cipher[2:]

# print(cipher1[32:])
secret = b''
for i in range(1,len(source_cipher)):
    finalcipher = xor_block(source_cipher[i],times2(E1)) + \
                  cipher1[16:32] + \
                  xor_block(source_cipher[0],bytearray(b'\x10'*15+long_to_bytes(15+i)))
    secret += (talk2(finalnonce,finalcipher,finaltag))
secret = secret[::-1]
print(secret)

r.recvuntil("[-] ")
r.sendline("3")
r.recvuntil("[-] ")
r.sendline(b64encode(secret))
r.recvuntil("flag: ")
flag = r.recvline(False)
print(flag)
# context(log_level='debug')
# r.interactive()
r.close()
```

## 方法总结

本题核心是 OCB nonce 复用下的 offset 复原和认证绕过。OCB 的每个块会用由 $E(\text{nonce})$ 派生出的 offset 做异或；nonce 可控且可复用时，可以通过构造明文/密文关系恢复这些 offset，并伪造后续需要的中间块。

识别信号是：服务允许重复 nonce、加密 oracle 和解密 oracle 同时存在、tag 可由已知 PMAC 关系调整、解密返回的 padding 行为能泄露 secret 字节。

复现时先用特殊明文恢复 $E(\text{nonce})$，再实现 OCB 中的 `times2/times3/back_times2` 和 `PMAC` 计算，构造 `from baby` 与 `from admin` 之间的 tag 差分。最后把 padding oracle 查询得到的 secret 逆序拼回，提交给菜单项 3 获取 flag。
