查阅RISC-V手册的目录, 你发现RV32I在哪一章进行介绍? 尝试在该章节中查阅RV32I的相关内容, 回答下列问题:

- PC寄存器的位宽是多少?
- GPR共有多少个? 每个GPR的位宽是多少?
- `R[0]`和sISA的`R[0]`有什么不同之处?
- 指令编码的位宽是多少? 指令有多少种基本格式?
- 在指令的基本格式中, 需要多少位来表示一个GPR? 为什么?
- `add`指令的格式具体是什么?
- `addi`指令的格式具体是什么?
- `jalr`指令的格式具体是什么?
- 还有一种基础指令集称为RV32E, 它和RV32I有什么不同?
- 为了了解RISC-V对存储器的若干约定, 你需要阅读RISC-V手册第1.4节的第一段, 从ISA的层面了解存储器的规格, 尤其是宽度的定义.
- 阅读RISC-V手册第1.1节，了解"hart"的概念——什么是hart？它和core、software thread有什么区别？

---

## 回答

> 依据：RISC-V 非特权架构手册（riscv-spec.pdf），RV32I 位于 **第 2 章 Base Instruction Sets → 2.1 节**（手册第 27 页）。

### PC 寄存器的位宽是多少？

**32 位。**

手册 2.1.1 节（第 27 页）Figure 1 和正文：对于 RV32I，XLEN=32。`pc` 是 XLEN 位宽的非特权寄存器，存放当前指令的地址。

### GPR 共有多少个？每个 GPR 的位宽是多少？

**32 个通用寄存器（x0–x31），每个 32 位。**

手册第 27 页原文：
> "For RV32I, the 32 **x** registers are each 32 bits wide, i.e., XLEN=32."

### `R[0]` 和 sISA 的 `R[0]` 有什么不同之处？

| | RISC-V 的 x0（R[0]） | sISA 的 r0（R[0]） |
|---|---|---|
| 性质 | 硬连线零寄存器（hardwired to 0） | 普通通用寄存器 |
| 写入 | 写入被忽略，值始终为 0 | 可写入，可保存任意值 |
| 读取 | 始终返回 0 | 返回最近写入的值 |
| 用途 | 编码伪指令（NOP、MV）、HINT、丢弃结果 | 可存放循环上界等数据值 |

手册第 27 页：**"Register x0 is hardwired with all bits equal to 0."**

sISA（教学简化 ISA，见 [`计算10以内的奇数之和.md`](obsidian://open?vault=notes&file=%E8%AE%A1%E7%AE%9710%E4%BB%A5%E5%86%85%E7%9A%84%E5%A5%87%E6%95%B0%E4%B9%8B%E5%92%8C)）中 r0 是 4 个通用寄存器之一，可被 `li r0, 11` 赋值为 11 并用于 `bner0` 比较。

### 指令编码的位宽是多少？指令有多少种基本格式？

- **位宽：32 位（固定长度）**

手册 2.1.2 节（第 28 页）：
> "In the base RV32I ISA, there are four core instruction formats (R/I/S/U)... All are a fixed 32 bits in length."

- **基本格式：4 种核心格式 + 2 种变体 = 共 6 种**

| 格式     | 用途           |
| ------ | ------------ |
| R-type | 寄存器-寄存器操作    |
| I-type | 寄存器-立即数操作    |
| S-type | 存储操作         |
| U-type | 高位立即数操作      |
| B-type | 条件分支（S 的变体）  |
| J-type | 无条件跳转（U 的变体） |

手册第 29 页说明 B/J 是 "two variants of the instruction formats based on the handling of immediates"。

### 在指令的基本格式中，需要多少位来表示一个 GPR？为什么？

**5 位。** GPR 共 32 个，2^5 = 32，恰好用 5 位唯一编码全部寄存器。

手册第 29 页格式图中 `rs1`、`rs2`、`rd` 均占 5 位。手册注释也提到解码寄存器标识符通常在实现的关键路径上，因此将所有寄存器字段固定在所有格式中的相同位置。

### `add` 指令的格式具体是什么？

`ADD` 使用 **R-type 格式**（手册 2.1.4.2 节，第 32 页）：

```
 31     25 | 24   20 | 19   15 | 14  12 | 11    7 | 6     0
┌──────────┬─────────┬─────────┬────────┬─────────┬─────────┐
│  funct7  │   rs2   │   rs1   │ funct3 │   rd    │ opcode  │
│ 0000000  │ 5 bits  │ 5 bits  │  000   │ 5 bits  │ 0110011 │
└──────────┴─────────┴─────────┴────────┴─────────┴─────────┘
  7 bits     5 bits    5 bits   3 bits    5 bits    7 bits
```

- **opcode** = `0110011`（OP）
- **funct3** = `000`（ADD）
- **funct7** = `0000000`（SUB 为 `0100000`）
- 语义：`R[rd] = R[rs1] + R[rs2]`，溢出忽略，仅保留低 XLEN 位

### `jalr` 指令的格式具体是什么？

`JALR` 使用 **I-type 格式**（手册 2.1.4.4 节 "Unconditional Jump Instructions"）：

```
 31         20 | 19    15 | 14  12 | 11    7 | 6       0
┌──────────────┬──────────┬────────┬──────────┬──────────┐
│  imm[11:0]   │   rs1    │ funct3 │    rd    │  opcode  │
│  12 bits     │  5 bits  │  000   │  5 bits  │ 1100111  │
└──────────────┴──────────┴────────┴──────────┴──────────┘
```

- **opcode** = `1100111`
- **funct3** = `000`
- **rs1**（位[19:15]）：基地址寄存器
- **rd**（位[11:7]）：目标寄存器，保存返回地址（pc+4）；设为 x0 则丢弃返回地址
- **imm[11:0]**（位[31:20]）：12 位有符号偏移量

**功能**：间接跳转并链接（Jump And Link Register）。

语义：
```
t   = pc + 4
pc  = (R[rs1] + sign_extend(imm)) & ~1   // 强制最低位为 0
R[rd] = t
```

目标地址由 rs1 + 有符号立即数决定，并强制最低位清零（保证 2 字节对齐）。

**典型用法**：

| 用法 | 汇编 | 说明 |
|------|------|------|
| 函数返回 | `jalr x0, x1, 0` | rs1=ra，rd=x0，等价于 `ret` |
| 间接调用 | `jalr x1, x5, 0` | 跳转到 x5 指向的函数，ra 保存返回地址 |
| 带偏移跳转 | `jalr x0, x2, 8` | 跳转到 sp+8 |

### `addi` 指令的格式具体是什么？

`ADDI` 使用 **I-type 格式**（手册 2.1.4.1 节，第 31 页）：

```
 31         20 | 19    15 | 14  12 | 11    7 | 6       0
┌─────────────┬──────────┬────────┬──────────┬──────────┐
│  imm[11:0]  │   rs1    │ funct3 │    rd    │  opcode  │
│  12 bits    │  5 bits  │  000   │  5 bits  │ 0010011  │
└─────────────┴──────────┴────────┴──────────┴──────────┘
  12 bits        5 bits    3 bits    5 bits      7 bits
```

- **opcode** = `0010011`（OP-IMM）
- **funct3** = `000`（ADDI）
- **rs1**（位[19:15]）：源寄存器
- **rd**（位[11:7]）：目标寄存器
- **imm[11:0]**（位[31:20]）：12 位立即数，有符号扩展到 XLEN 位

**功能描述**（手册原文）：

> "ADDI adds the sign-extended 12-bit immediate to register rs1. Arithmetic overflow is ignored and the result is simply the low XLEN bits of the result. ADDI rd, rs1, 0 is used to implement the MV rd, rs1 assembler pseudoinstruction."

语义：`R[rd] = R[rs1] + sign_extend(imm[11:0])`，溢出忽略，仅保留低 XLEN 位。

**举例**：`addi x5, x6, 7` → `x5 = x6 + 7`

**与 ADD 的对比**：

| 特征 | ADD | ADDI |
|------|-----|------|
| 格式 | R-type | I-type |
| opcode | `0110011` | `0010011` |
| 操作数 | rs1, rs2 | rs1, 立即数 |
| 用途 | 寄存器-寄存器加法 | 寄存器-立即数加法 |

### RV32E 和 RV32I 有什么不同？

**唯一区别：通用寄存器从 32 个减少到 16 个（x0–x15）。**

手册 2.3 节（第 47 页）：
> "RV32E and RV64E are reduced versions of RV32I and RV64I, respectively: **the only change is to reduce the number of integer registers to 16.**"

其余完全一致：指令编码相同（x16–x31 编码 reserved）、指令语义相同、兼容的标准扩展也兼容 RV32E。设计动机：在小规模 RV32I 核心中高 16 个寄存器约占 1/4 面积，去除后可节省约 25% 面积和功耗，面向嵌入式微控制器。

### RISC-V 对存储器有哪些约定？（手册第 1.4 节，第 21 页）

**地址空间**（第一段）：

> "A RISC-V hart has a single byte-addressable address space of 2^XLEN bytes for all memory accesses, where a byte is 8 bits. The memory address space is circular, so that the byte at address 2^XLEN−1 is adjacent to the byte at address zero. Accordingly, memory address computations done by the hardware ignore overflow and instead wrap around modulo 2^XLEN."

| 要点   | 说明                                       |
| ---- | ---------------------------------------- |
| 地址空间 | 每个 hart 拥有一个字节寻址（byte-addressable）的地址空间  |
| 大小   | **2^XLEN 字节**（RV32I 中 XLEN=32，即 4 GiB）   |
| 字节定义 | **1 byte = 8 bits**                      |
| 地址循环 | 地址空间是 circular（循环的），地址 2^XLEN−1 与地址 0 相邻 |
| 溢出行为 | 硬件地址计算**忽略溢出**，按模 2^XLEN 回绕              |

**存储器宽度定义**（第二段）：

> "A word of memory is defined as 32 bits (4 bytes). Correspondingly, a halfword is 16 bits (2 bytes), a doubleword is 64 bits (8 bytes), and a quadword is 128 bits (16 bytes). Furthermore, a nibble is 4 bits."

| 单位 | 宽度 | 字节数 |
|------|------|--------|
| nibble | 4 bits | 0.5 B |
| byte | 8 bits | 1 B |
| halfword | 16 bits | 2 B |
| **word** | **32 bits** | **4 B** |
| doubleword | 64 bits | 8 B |
| quadword | 128 bits | 16 B |

> 注意：RISC-V 中 **word 固定为 32 bits**，与 XLEN 无关。这与 x86 中 "word=16 bits"、ARM 中 "word 取决于位宽" 的惯例不同。

### 什么是 hart？（手册第 1.1 节）

**hart** = **HAR**dware **T**hread（硬件线程），是 RISC-V 架构中最小的独立执行单元。

手册原文：

> "A RISC-V compatible core might support multiple RISC-V-compatible hardware threads, or harts, through multithreading."

| 概念 | 说明 |
|------|------|
| hart | 硬件线程，拥有独立的 PC 和寄存器文件（x0–x31），可独立取指、译码、执行。从软件视角看，一个 hart 就是一个独立的 CPU |
| core | 物理核心，一个 core 可通过多线程（SMT）同时运行多个 hart |
| software thread | 操作系统层面的调度抽象；hart 上运行的软件感知不到上下文切换，始终觉得自己独占整条流水线 |

**关键区分**：

- **hart ≠ core**：一个多线程核心可以有多个 hart。"RISC-V compatible core might support multiple ... harts"
- **hart ≠ software thread**：hart 是硬件执行资源，软件线程是 OS 调度概念。hart 是硬件层面的最小执行实体，它给上层的软件线程提供了一个"看起来像独立 CPU"的执行环境

> 简记：**hart 是硬件能独立执行一条指令流的最小实体。** 之前存储器章节里 "A RISC-V hart has a single byte-addressable address space..." ，就是在说每个 hart 都拥有自己独立的 2^XLEN 字节地址空间。


