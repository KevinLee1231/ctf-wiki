# GreyCTF2024 Verilog Count WP

## 题目简述

服务要求提交一份 Verilog 计数器：不能依赖高级算术黑盒，而要用全加器构造 32 位加法器，使 `result` 在每个时钟上升沿加一。提交内容需通过测试平台模拟后才会返回 flag。

## 解题过程

一位全加器满足：

$$
S=A\oplus B\oplus C_{in}
$$

$$
C_{out}=(A\land B)\lor(C_{in}\land(A\oplus B))
$$

将 32 个全加器的进位首尾相接，就得到 ripple-carry adder。完整核心实现如下：

```verilog
module my_full_adder(input A, B, CIN, output S, COUT);
    assign S = A ^ B ^ CIN;
    assign COUT = (A & B) | (CIN & (A ^ B));
endmodule

module adder(
    input [31:0] A, B,
    input C0,
    output [31:0] S
);
    wire [32:0] carry;
    assign carry[0] = C0;

    genvar i;
    generate
        for (i = 0; i < 32; i = i + 1) begin : ripple
            my_full_adder fa(A[i], B[i], carry[i], S[i], carry[i + 1]);
        end
    endgenerate
endmodule

module counter(input clk, output reg [31:0] result);
    wire [31:0] next;
    adder add_one(result, 32'd1, 1'b0, next);

    initial result = 0;
    always @(posedge clk)
        result <= next;
endmodule
```

把源码按服务要求进行 Base64 编码并提交，测试平台观察到计数序列正确后返回：

```text
grey{c0un71n6_w17h_r1pp13_4ddr5}
```

## 方法总结

多位加法器的本质是逐位传播进位。使用 `generate` 只是消除 32 次机械重复，不改变电路结构；计数器本身则把当前输出与常量 1 相加，并在时钟上升沿锁存结果。需要同时注意组合逻辑与时序逻辑的边界。
