查阅RISC-V手册的目录, 你发现RV32I在哪一章进行介绍? 尝试在该章节中查阅RV32I的相关内容, 回答下列问题:

- PC寄存器的位宽是多少?
- GPR共有多少个? 每个GPR的位宽是多少?
- `R[0]`和sISA的`R[0]`有什么不同之处?
- 指令编码的位宽是多少? 指令有多少种基本格式?
- 在指令的基本格式中, 需要多少位来表示一个GPR? 为什么?
- `add`指令的格式具体是什么?
- 还有一种基础指令集称为RV32E, 它和RV32I有什么不同?

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

| 格式 | 用途 |
|---|---|
| R-type | 寄存器-寄存器操作 |
| I-type | 寄存器-立即数操作 |
| S-type | 存储操作 |
| U-type | 高位立即数操作 |
| B-type | 条件分支（S 的变体） |
| J-type | 无条件跳转（U 的变体） |

手册第 29 页说明 B/J 是 "two variants of the instruction formats based on the handling of immediates"。

### 在指令的基本格式中，需要多少位来表示一个 GPR？为什么？

**5 位。** GPR 共 32 个，2^5 = 32，恰好用 5 位唯一编码全部寄存器。

手册第 29 页格式图中 `rs1`、`rs2`、`rd` 均占 5 位。手册注释也提到解码寄存器标识符通常在实现的关键路径上，因此将所有寄存器字段固定在所有格式中的相同位置。

### `add` 指令的格式具体是什么？

`ADD` 使用 **R-type 格式**（手册 2.1.4.2 节，第 32 页）：

```
 31          25 24      20 19      15 14     12 11       7 6          0
┌──────────────┬──────────┬──────────┬─────────┬───────────┬────────────┐
│   funct7     │   rs2    │   rs1    │ funct3  │    rd     │   opcode   │
│  0000000     │  5 bits  │  5 bits  │  000    │  5 bits   │  0110011   │
└──────────────┴──────────┴──────────┴─────────┴───────────┴────────────┘
   7 bits         5 bits     5 bits    3 bits     5 bits       7 bits
```

- **opcode** = `0110011`（OP）
- **funct3** = `000`（ADD）
- **funct7** = `0000000`（SUB 为 `0100000`）
- 语义：`R[rd] = R[rs1] + R[rs2]`，溢出忽略，仅保留低 XLEN 位

### RV32E 和 RV32I 有什么不同？

**唯一区别：通用寄存器从 32 个减少到 16 个（x0–x15）。**

手册 2.3 节（第 47 页）：
> "RV32E and RV64E are reduced versions of RV32I and RV64I, respectively: **the only change is to reduce the number of integer registers to 16.**"

其余完全一致：指令编码相同（x16–x31 编码 reserved）、指令语义相同、兼容的标准扩展也兼容 RV32E。设计动机：在小规模 RV32I 核心中高 16 个寄存器约占 1/4 面积，去除后可节省约 25% 面积和功耗，面向嵌入式微控制器。