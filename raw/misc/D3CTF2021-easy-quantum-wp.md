# easyQuantum

## 题目简述

本题给出 `cap.pcapng`，网络流量中传输的是 Python pickle 序列化后的量子态、Bob 测量基、Alice 判断结果和最终密文。通过包长度规律和 `numpy`/pickle 头部可以识别出这是简化版 BB84 量子密钥分发流程，开头的 `312` 是约定密钥长度。

题目附件包含一份量子通信流量和求解脚本所需的数据形态：PCAP 中可提取 pickle/numpy 序列化后的量子状态、Bob 的测量基、Alice 的筛选结果和密文。题目提供了调试信息形式的量子状态向量，等价于知道 Alice 发送的量子；据此结合 Bob 的测量基和 Alice 的筛选结果恢复密钥，再与等长密文按位异或得到 flag。

## 解题过程

题目源码和流量生成逻辑见 [`zhouweitong3/d3ctf_easyQuantum`](https://github.com/zhouweitong3/d3ctf_easyQuantum)。下文保留了仓库和 PDF 中的关键数据形态：pickle/numpy 序列化、BB84 状态向量、Bob 测量基、Alice 筛选结果和最终异或解密方式。

用 Wireshark 打开 `cap.pcapng` 后，能看到正常 TCP 三次握手和四次挥手，需要重点分析中间的数据传输。部分数据包中出现 `numpy` 字符串，同时有类似 pickle protocol 4 的固定头部，因此可以先尝试用 pickle 反序列化 payload：

```python
import pickle
import pyshark

cap = pyshark.FileCapture("cap.pcapng")
data = cap[3].data.data.binary_value
print(pickle.loads(data))
```

开头反序列化得到 `312`，对应约定密钥长度。后续数据包长度呈现固定组：`436-(ACK)-90-(ACK)-90-(ACK)` 或 `436-(ACK)-81-(ACK)`。拿一组反序列化后，可看到四个量子状态向量、Bob 的测量基数组和 Alice 的判断结果，例如：

```text
Data: [
  array([0.70710678+0.j, 0.70710678+0.j]),
  array([0.+0.j, 1.+0.j]),
  array([1.+0.j, 0.+0.j]),
  array([0.70710678-8.65956056e-17j, -0.70710678+8.65956056e-17j])
]
Bob bases: [0, 0, 1, 1]
Alice judge: [0, 1, 0, 1]
```

题目中的 QKD 指量子密钥分发。结合“状态向量传递量子、两个数组依次传递 Bob 测量基和 Alice 筛选结果”的流程，可以确认这是简化版 BB84。空数据包可以理解为 Eve 测量导致 Bob 没有收到有效量子。题目提供的 debug info 等价于泄露 Alice 发送的量子状态，因此无需真正窃听量子信道，只要按状态向量、Bob 测量基和 Alice 判断结果恢复筛选后的 key。

这里使用的是常见量子态向量表示：`|phi> = alpha|0> + beta|1>` 对应数组 `[alpha, beta]`。因此 `[1, 0]` 表示 `|0>`，`[0, 1]` 表示 `|1>`，`[0.707..., 0.707...]` 和 `[0.707..., -0.707...]` 分别对应 Hadamard 基下的两种状态。

为了方便处理，可先用过滤器只保留带数据的 TCP 包：

```text
tcp.flags == 0x018
```

脚本思路如下：反序列化每组量子状态，用状态向量恢复量子门；按 Bob 的测量基测量；Alice 判断为真的位置拼入 key；最后用 key 与密文按位异或。

```python
import pickle
import pyshark
import qiskit
from bitstring import BitArray

cap = pyshark.FileCapture("cap.pcapng")
key = ""

def decrypt_msg(enckey: BitArray, msg: BitArray):
    res = BitArray()
    for i in range(msg.len):
        res.append("0b" + str(int(enckey[i] ^ msg[i])))
    return res

def recv_quantum(quantum_state):
    quantum = [qiskit.QuantumCircuit(1, 1) for _ in range(4)]
    for i in range(4):
        real_a = quantum_state[i][0].real
        real_b = quantum_state[i][1].real
        if real_a == 1.0 and real_b == 0.0:
            continue
        if real_a == 0.0 and real_b == 1.0:
            quantum[i].x(0)
        elif real_a > 0.7 and real_b > 0.7:
            quantum[i].h(0)
        else:
            quantum[i].x(0)
            quantum[i].h(0)
    return quantum

def measure(receiver_bases, quantum):
    for i in range(4):
        if receiver_bases[i]:
            quantum[i].h(0)
        quantum[i].measure(0, 0)
        quantum[i].barrier()
    backend = qiskit.Aer.get_backend("statevector_simulator")
    return qiskit.execute(quantum, backend).result().get_counts()

def append_key(qubits, bases, compare_result):
    global key
    measure_result = measure(bases, qubits)
    for i in range(4):
        if compare_result[i]:
            key += list(measure_result[i].keys())[0]

key_len = pickle.loads(cap[0].data.data.binary_value)
i = 1
while i < 567:
    if int(cap[i + 1].data.len) == 15:
        i += 2
        continue
    quantum_state = pickle.loads(cap[i].data.data.binary_value)
    bob_bases = pickle.loads(cap[i + 1].data.data.binary_value)
    alice_judge = pickle.loads(cap[i + 2].data.data.binary_value)
    append_key(recv_quantum(quantum_state), bob_bases, alice_judge)
    i += 3

key = key[:key_len]
msg = BitArray(pickle.loads(cap[567].data.data.binary_value))
plaintext = decrypt_msg(BitArray("0b" + key), msg)
print(plaintext.tobytes())
```

得到Flag:

  d3ctf{y1DcuFuYwCgRfX33uT1lgSy27jYIsF4i}

当然，也可以不调用 qiskit，直接建立“状态向量 + Bob 测量基 + Alice 判断结果”到 bit 的映射来恢复 key。

## 方法总结

- 核心技巧：从 PCAP 中识别 pickle 序列化的量子态数据，按 BB84 流程恢复筛选后的密钥，再作为流密码解密。
- 识别信号：TCP 数据包中出现 `numpy`、pickle 协议头、固定长度的“量子态数组/测量基/判断结果”三元组，以及密文长度等于密钥长度。
- 复用要点：量子题不一定要真正跑量子模拟；如果状态向量、测量基和筛选结果都在流量里，直接建立状态到 bit 的映射也能恢复密钥。

