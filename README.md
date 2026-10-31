# BUAA-CO-CPU-alone
（北航计组）应该是第一个无tunnel无submodule纯wire的CPU（logisim）

(BUAA-CO) This may be the first CPU with pure wires that has no tunnel and no submodule (logisim)

如下即为电路整体外观，用的新版logisim（因为好看~），旧版（2.7.1）也可运行（需要注意一下与或门连线可能会错位）

<img width="1279" height="763" alt="屏幕截图 2026-10-29 231549" src="https://github.com/user-attachments/assets/e2a1189d-22e8-4890-9164-5ec526a97f6d" />

<h1 align="center"> myCPU设计文档 </h1>

本人将CPU按设计顺序分成了PC、NPC、EXT、IFU、ALU、GRF、DM、CTRL共8个模块

**1. PC**
<img width="1117" height="538" alt="image" src="https://github.com/user-attachments/assets/7a15920d-39f5-4e49-82d8-e8ab8d22f3bb" />

PC输入是从NPC产生的输出信号导入，
输出是IFU读取所需指令的地址，同时也作为NPC的一个输入。

PC的本质是一个寄存器，因为PC涉及到**初始状态为0x3000**，所以这里用非常经典的**常数-异或-异或**结构的给寄存器**赋初始值**的结构。

**2. NPC**
<img width="1380" height="1090" alt="image" src="https://github.com/user-attachments/assets/231fd2e6-93ba-4e05-aebe-809b33208ea7" />


输入有**PC传值**、16位立即数**imm16**、26位立即数**imm26**（用于jal）、32位寄存器内部地址数据**ra32**（用于jr），控制信号**JR**、**Jump**、**Branch_and_Zero**。
输出有**NextPC**、**PC+4**（jal的地址注入）

NPC是纯逻辑电路，根据当前的输入直接得出下一步输出，即我的PC下一步应该走到哪里呢？

根据规定的机器码操作流程，NPC设计的内部操作有
- PC信号正常+4
- imm16有符号拓展32位左移两位+PC+4
- imm26低位补两个零，高四位接PC高四位

并用JR、Jump、Branch_and_Zero拼成的3位信号和MUX构成类似于独热编码的控制信号控制NPC从哪一支输出。

**3. EXT**
<img width="891" height="366" alt="image" src="https://github.com/user-attachments/assets/d421d9fb-33b5-4d8a-86d3-dadb4ebc3a54" />

输入是16位立即数、控制信号ExtOp，
输出是32位的扩展数

由MUX控制，若ExtOp亮起，则选择有符号拓展


**4. IFU**
<img width="1255" height="504" alt="image" src="https://github.com/user-attachments/assets/9648d6a8-72fd-42c3-82d9-666b8346eb85" />

输入是32位PC（告诉IFU应该读取的指令的地址）
输出是32位指令

用**ROM**实现保存机器码指令，右键它可导入测试数据

因为PC在输出前经过了一系列的加减乘除移位操作，这里输入后先**减去0x3000**，后**右移两位**再用Bit Selector取出**低16位**

**5. ALU**
<img width="1138" height="769" alt="image" src="https://github.com/user-attachments/assets/45a00d6b-4604-40c2-ba9f-a4c18a1120ed" />

输入是两个32位操作数、ALUctrl控制信号，
输出是Zero判零信号、result

提供加、减、或、移位操作，根据MUX和ALUctrl控制信号控制得到所需操作的输出
- add: A add B
- sub: A sub B
- ori: A or B
- lui: B shift 16 （一开始用的A，找了很久bug）
- beq: Zero = (A sub b == 0) 

**6. GRF**
<img width="1929" height="1105" alt="image" src="https://github.com/user-attachments/assets/04b9dfd6-1d2e-470e-ae41-b15c57b91a5c" />

输入是write_enable、write_data、三个选位信号A1、A2、A3
输出是32位读出数据RD1、RD2

由DMX、write_enable和A3决定哪个寄存器能写，由MUX和A1、A2决定读出哪两个寄存器的值

**注意0号寄存器即使能写入也一直保持0**

**7. DM**
<img width="1011" height="708" alt="image" src="https://github.com/user-attachments/assets/ec7d07ea-4633-48a1-9e13-2196af3303ca" />

输入是32位Addr，write_data(WD),1位write_enable(WE)
输出是32位RAM的data值

因为RAM的数据容量也是有限制的，所以需要**将Addr截取**2-17（共16位）作为地址的十六进制索引

**8. CTRL(Controller)**
<img width="1992" height="1332" alt="image" src="https://github.com/user-attachments/assets/6235461e-46a5-4e35-88ff-5097ae8c8d0b" />

输入是从32位机器码接下来的高6位Opcode、低6位Func
输出是RegDst、RegWrite、MemToReg（无用）、MemWrite、MemRead、ALUSrcI、beq、nop（无用）、jal、ExtOp、四位ALUctrl

通过**与或门阵列**捕获到这条机器码所代表的指令操作头

如：
- ADD：add RD, RS, RT
编码：000000 /**/ 100000
描述：GPR[rd] = GPR[rs] + GPR[rt]
- SUB：sub RD, RS, RT
编码：000000 /**/ 100010
描述：GPR[rd] = GPR[rs] - GPR[rt]
- ORI：ori RT, RS, IMM
编码：001101 /**/
描述：GPR[rt] = GPR[rs] | zero_extend(IMM)
- LW：lw RT, OFFSET(RS)
编码：100011 /**/
描述：GPR[rt] = MEM[GPR[rs] + sign_ext(OFFSET)]
- SW：sw RT, OFFSET(RS)
编码：101011 /**/
描述：MEM[GPR[rs] + sign_ext(OFFSET)] = GPR[rt]
- BEQ：beq RS, RT, LABEL
编码：000100 /**/
描述：if GPR[rs] == GPR[rt], PC = PC + 4 + sign_ext(LABEL); else PC = PC + 4;
- lui：lui RT, IMM
编码：001111 /**/
描述：GPR[rt] = sign_ext(IMM) << 16;
- nop：nop
编码：000000 /**/ 000000

最后把需要用到ALU原件的指令信号收集起来用Priority Encode进行一个超级拼装，得到ALUCtrl，在ALU体现为需要进行的运算的选择信号。

**9. 顶层设计**
主要讲一下注意事项：
- NPC的Branch_and_Zero输入要把CTRL的beq和ALU的Zero信号与一下
- 用分线器把IFU的32位输出把所需的信号段全部拆解下来
- GRF的A3根据RegDst和jump（固定写入31号）信号选择要写入的寄存器地址
- GRF的WD根据MemRead和jump(固定写入PC+4)信号选择要写入的数据
- 实现jr指令时ra32由GRF的RD1输出与上32位0xfffffffc得到（低两位置0）
- ALUSrcI选择ALU的B操作数来源（默认RD2,否则ext32）
- DM的Add来源于ALU的计算结果，MemData来源于RD2
- 其他的信号按部就班的用tunnel连上即可

**10. 最终效果**
<img width="2311" height="1374" alt="image" src="https://github.com/user-attachments/assets/d4ee3be0-be7c-4b04-9103-29acd3f038af" />


好了，上面是本人花了两天从完全未知、畏惧到搭建完成的CPU的设计文档。

**11. other**
（由于本人更喜欢连线，所以专门又花了两小时手搓了这个无tunnel无submodule的纯wire的CPU）

<h2 align="center"> 思考题 </h2>

**1.** 状态存储:PC、GRF、DM、IFU
状态转移：NPC、CTRL、EXT、ALU.

**2.** 整体合理，是"分离指令/数据存储 + 集中寄存器堆"的哈佛单周期结构，应该是标准的做法。

IM 用 ROM：运行期指令只读（无自修改代码），ROM 不需要写口，面积小、内容固化、结果可复现；Logisim 的 ROM 是组合读，地址一变输出即变，天然符合单周期取指时序；
DM 用 RAM：lw/sw 必须有可写且跨指令保持的存储；RAM 比 32 个独立寄存器阵列省资源；组合读 + 时钟写正好配单周期；
GRF 用 Register：寄存器堆要求"两个异步读口 + 一个时钟写口 + 同拍读旧写新"，用触发器阵列 + 译码/多路选择最直接，$0 硬连 0 也好做。

**3.** 没有，教程做得很好很合理

**4.** nop 的编码就是 0x00000000，与 sll $0,$0,0 同编码。译码器本来就无法也不该区分二者。
这条编码译出的控制信号天然全 0：RegWrite=0、MemWrite=0、MemRead=0、Branch=0，ALU 算出什么都不写回。
于是数据通路的默认行为就是 PC ← PC+4、所有状态不变

**5.** 测试包含 ori、lui、add、sw、lw、beq 六条。sub、nop完全没有出现
| 指令 | 已覆盖 | 未覆盖（具体不足） |
| :--- | :--- | :--- |
| **ori** | `rs=$0` 与 `rs≠0`；`imm 0xffff` 参与运算 | 真正验证零扩展的用例（`ori $a2,$0,-456` 被注释掉了）；写 `$0`；`imm` 高位 `0x8000` |
| **lui** | `imm` 符号位为 0 与 1 | `imm=0`、`imm=0x8000`、目标为 `$0`；未验证移位后低位为 0 |
| **add** | 正+正、正+负、负+负 | 溢出（`0x7fffffff+1`）、进位（`0xffffffff+1`）、`0+0`、写 `$0`、结果为 0（`Zero=1`） |
| **sw** | 基址 0、偏移 0/4/…/24 | 非零基址、负偏移、高位地址、同址重复写、写 `$0` 作为数据 |
| **lw** | 偏移 0 与 12 | 读未写过的地址、负偏移、高位地址、与相邻 `sw` 的先后关系 |
| **beq** | 前向 taken / not taken 各一次 | 后向分支、偏移 0（自跳）、相等且为 0 或负数（如 `beq $0,$0`）、误跳对后续 PC 的连锁影响 |

- 扩展方式假覆盖：lui $a3,0xffff 得 0xffff0000，再 ori $a3,$a3,0xffff。若 ori 错用符号扩展得 0xffffffff，用零扩展也是 0xffffffff —— 两种实现结果完全相同，这条根本测不出 ExtOp/零扩展的对错。
- 符号扩展从未真正验证：lw/sw 的 offset 全是 0～40 的非负小值，beq 的 label 也是小的正偏移；sign_ext(imm16) 的高位复制、负偏移、0x8000 边界都没测。而 EXT、NPC 的 imm16 支路、DM 地址计算都依赖它。
- 地址空间覆盖极窄：基址恒为 $t0=0，偏移 0～40，只触到 DM 地址译码的低 6 位。你用 Addr[17:2] 截断，任何高位地址/别名 bug（截断错位、把字地址当字节地址）都完全不可见。
