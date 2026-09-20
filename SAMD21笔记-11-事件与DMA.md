# 第 11 章 事件与传输：EVSYS 与 DMAC

> **芯片**：ATSAMD21G18A（Cortex-M0+ @48 MHz、256 KB Flash、32 KB SRAM、48 脚 / 38 个 I/O）
> **本章目标**：搞清两件事——外设怎么**不经过 CPU** 就通知另一个外设（EVSYS），数据怎么**不经过 CPU** 就搬家（DMAC）。这两件事是同一套配合的两半：事件决定"什么时候动"，DMA 决定"搬什么"。
> **学习顺序**：为什么需要它们 → 事件是什么（发生器 / 通道 / 用户）→ 哪些外设有事件 → 路径与边沿 → 翻译成寄存器 → EVSYS 寄存器 → DMA 是什么（通道 / 描述符 / 触发）→ DMAC 寄存器 → 配置顺序 → 两个例程 → 初始化清单 → 思维模型 → 常见问题 → 速查 → 自测。
> **本章不写**：时钟源细节（第 5 章）、中断向量与 NVIC（第 12 章）、ADC 本体配置（第 10 章，本章只引用）。代码里用到它们时只给最少的必要配置。

## 11.0 三层演进：从轮询到硬件自动联动

同一个需求——"ADC 每采到一个值就存进数组"——有三代实现方式，理解它们的差别就理解了 EVSYS 和 DMAC 存在的理由。**轮询**：CPU 在循环里不停读标志位，看到"转换完成"就读结果、存数组，全程陪着什么都干不了。**中断**：ADC 转换完成后拉中断线，CPU 进中断服务程序（ISR）搬一趟再返回，不用干等，但每次都要"进中断、出中断"，数据终究还是 CPU 搬的。**事件 + DMA**：ADC 发出一个**事件**，事件经由 **EVSYS** 送到 **DMAC**，DMAC 自己把数据从 ADC 的结果寄存器搬到数组，CPU 不在场，只在这批数据搬完后处理一次。

| 层 | 谁发现"好了" | 谁搬数据 | 每个采样的 CPU 开销 | 1 Msps 时的 CPU 占用 | 睡眠时 |
|---|---|---|---|---|---|
| 轮询 | CPU 反复读标志 | CPU | 十几条指令，且必须一直等 | 基本吃满 | 不行 |
| 中断 | 外设拉中断线，CPU 在 ISR 里判断 | CPU | 进出 ISR + 读写 ≈ 20~30 周期 | 约 50%~100% | 会把 CPU 唤醒（费电） |
| 事件 + DMA | EVSYS 直连外设 | DMAC | 0（只在头尾各处理一次） | ≈0 | 可以（SleepWalking，第 6 章） |

上表的周期数是**量级估算，不是手册数据**（Cortex-M0+ 没有搬运指令，搬一个字节就是"读—写—改指针—判循环"，中断方式还要加上约 16 个周期的压栈弹栈）。延迟那一列也值得记：中断的进出时间不固定、还可能被更高优先级挤掉，而事件的传播延迟是固定的几个时钟、不会被打断。结论很直白：**采样率一上去，软件搬运就是瓶颈**——而 DMAC 搬 2 MB/s 对 AHB 总线很轻松（48 MHz、32 位总线，理论峰值 192 MB/s）。所以第三层不是"更优雅"，而是"还能不能跑得动"的区别。

## 11.1 EVSYS 是什么：外设之间的连线

事件系统（EVSYS，Event System）只做一件事：**把一个外设"发生了某件事"这个信号送到另一个外设**。发出事件的外设叫**事件发生器（generator）**，比如 ADC 的"结果就绪"、EIC 的"EXTINT3 检测到边沿"；响应事件的外设叫**事件用户（user）**，比如 DMAC 的某条通道、TCC 的某个计数器动作。

```text
发生器（谁产生事件）          通道（12 条连线）          用户（谁来响应）
EIC / TC / TCC             ┌── CH0 ──┐               TC / TCC
ADC / AC / DAC   ─────────▶│   ...   │──────────────▶ ADC / AC / DAC
RTC / DMAC CH0-3           └── CH11 ─┘               DMAC CH0-3
```

**事件通道（channel）** 有 12 条，一条通道同一时刻只能接**一个**发生器，但它的输出可以**广播给多个用户**。三条性质决定了后面所有配置：① **通道是连线，不是数据通路**——事件只表示"发生了"，不携带任何数据，ADC 采到的 12 位数值仍然躺在 `ADC.RESULT` 里，谁去读是另一回事；② **同一个外设可以既是发生器又是用户**，比如 TCC0 既能"溢出时发事件"，也能"收到事件时启动计数"；③ **每个用户自己挑听哪条通道**，所以同一个事件可以同时通知 DMAC（搬数据）和另一个外设（做动作），这是中断做不到的。

把两个模块并排看，分工就清楚了：

| | EVSYS | DMAC |
|---|---|---|
| 传递什么 | 一个"发生了"的脉冲 | 真正的数据字节 |
| 连接方式 | 外设 ↔ 外设（可广播） | 内存 ↔ 外设（点对点） |
| 回答的问题 | "什么时候搬？" | "搬什么、搬多少、搬到哪？" |
| 单独能干什么 | 让 TCC 溢出时自动启动一次 ADC 转换 | 软件触发搬一块内存 |

**只有"动作"没有"数据"时，EVSYS 一个就够**（TCC0 溢出 → ADC 启动转换，全程没有数据要搬）；**有数据要搬时两者配合**，事件负责"什么时候"，DMAC 负责"搬什么"。本章两个例程正好对应这两种组合。

## 11.2 EVSYS 的三段模型与配置顺序

要接通一条事件路径，一共要动三个地方，**顺序是手册定死的：先配用户，再配通道**（手册 24.6.2.1）。

| 步骤 | 在哪里配 | 回答什么问题 |
|---|---|---|
| ① 发生器侧开关 | 各外设自己的 `EVCTRL` | 这个外设要不要往外发事件？ |
| ② 用户 | `EVSYS.USER` | 哪个用户，听哪条通道？ |
| ③ 通道 | `EVSYS.CHANNEL` | 这条通道接哪个发生器、走哪条路、认哪个边沿？ |

为什么必须先 ② 后 ③？把通道想成一根线：先把用户接到线上（②），再让线通上信号源（③）；反过来做，通道已经开始产生事件了，而线上还没有用户接听。另一个容易写错的地方：**`USER.CHANNEL` 里写的是"通道号 + 1"**——想接通道 0，字段里要写 1，写 0 表示"不接任何通道"（手册 24.8.3）。

```c
/* 把 DMAC CH0 挂到通道 0 上；CHANNEL 字段要写通道号 + 1 */
EVSYS_REGS->EVSYS_USER = EVSYS_USER_USER(0x00) | EVSYS_USER_CHANNEL(1);

/* 通道 0 接发生器 0x42（ADC RESRDY）、走重同步路径、认上升沿；必须一次 32 位写完 */
EVSYS_REGS->EVSYS_CHANNEL = EVSYS_CHANNEL_CHANNEL(0)
                          | EVSYS_CHANNEL_EVGEN(0x42)
                          | EVSYS_CHANNEL_PATH_RESYNCHRONIZED
                          | EVSYS_CHANNEL_EDGSEL_RISING_EDGE;
```

这里的 `0x42`、`0x00` 都是**编号**，不是随手写的数，下一节讲怎么查。

## 11.3 哪些外设有事件：手册 Table 12-1 的 Events 列怎么读

整本手册第 12 章只有一张表（**Table 12-1 Peripherals Configuration Summary**），行是外设，列依次是：基地址、IRQ 号、AHB 时钟、APB 时钟、通用时钟、PAC 写保护、**Events（User / Generator）**、DMA。**Events 列的两个子列正好对应上一节的 ② 和 ③：**

| 子列 | 它的数填到哪 | 怎么用 |
|---|---|---|
| **User** | `EVSYS.USER` 的 `USER` 字段 | 这个外设作为用户时的编号——它当用户时，往 `USER` 里写几 |
| **Generator** | `EVSYS.CHANNEL` 的 `EVGEN` 字段 | 这个外设的事件编号——往 `EVGEN` 里写几 |

表里的数字是**十进制**（`0x42` 在表里写成 66），而 `EVGEN` 字段里填的是同一个数的十六进制。以本章要用到的几行为例：

| 外设 | Events: User | Events: Generator | DMA 列（另一条路，见 11.9） |
|---|---|---|---|
| ADC | 23: START、24: SYNC | 66: RESRDY、67: WINMON | 39: RESRDY |
| DMAC | 0-3: CH0-3 | 30-33: CH0-3 | — |
| EIC | — | 12-27: EXTINT0-15 | — |
| TC3 / TC4 / TC5 | 18 / 19 / 20: EV | 51 / 54 / 57: OVF；52-53 / 55-56 / 58-59: MC0-1 | 24-26 / 27-29 / 30-32 |
| TCC0 / TCC1 / TCC2 | 4-9 / 10-13 / 14-17 | 34-40 / 41-45 / 46-50 | 13-17 / 18-20 / 21-23 |
| AC（1 个模块、2 个比较器） | 25-26: SOC0-1 | 68-69: COMP0-1、70: WIN0 | — |
| DAC | 27: START | 71: EMPTY | 40: EMPTY |
| RTC | — | 1: CMP0/ALARM0、2: CMP1、3: OVF、4-11: PER0-7 | — |

对照手册 24.8.2 的 `EVGEN` 表可以验证：66 = 0x42 = ADC RESRDY，DMAC 的 30 = 0x1E = DMAC CH0。**Table 12-1 是十进制索引，`EVGEN` 表是十六进制值，指的是同一批编号。** 本器件（G18A）有两点要注意：表里的 **TC6/TC7（0x3C-0x41）和 TCC3（0x4D-0x53）这颗芯片没有**（G18A 只有 TC3-TC5、TCC0-TCC2，AC 也只有 2 个比较器，没有 COMP2/COMP3），Table 12-1 列的是整个 SAM D21/DA1 家族，不要照抄；**SERCOM 不在 Events 列里**——它的"收到一个字节 / 发送寄存器空"是 11.9 节讲的 DMAC 专用触发源（DMA 列里的 1: RX、2: TX），不经过 EVSYS，而它的帧错误、校验错误也不产生事件，只能用中断查（第 12 章）。

### 发生器侧的开关：事件输出默认是关的

编号只是"接线"，还要在**发生器那一侧**把事件输出打开——每个外设自己有 `EVCTRL` 寄存器（名字和位置各不相同）：

| 外设 | 使能位 | 含义 |
|---|---|---|
| ADC | `ADC.EVCTRL.RESRDYEO` | 结果就绪时发事件 |
| DAC | `DAC.EVCTRL.EMPTYEO` | 数据缓冲空时发事件 |
| AC | `AC.EVCTRL.COMPEO0/1`、`WINEO0` | 比较器翻转 / 窗口命中时发事件 |
| TC | `TC.EVCTRL.OVFEO`、`MCEO0/1` | 溢出 / 匹配捕获时发事件 |
| TCC | `TCC.EVCTRL.OVFEO`、`TRGEO`、`CNTEO`、`MCEO0-3` | 溢出 / 重触发 / 计数 / 匹配时发事件 |
| EIC | `EIC.EVCTRL.EXTINTEOx` | 某个 EXTINT 检测到时发事件 |
| RTC | `RTC.MODEn.EVCTRL` 的 `PEREO0-7`、`CMPEO0/1`、`OVFEO` | 周期 / 比较 / 溢出时发事件 |
| DMAC | `DMAC.CHCTRLB.EVOE` + 描述符 `BTCTRL.EVOSEL` | 一个 beat 或一个 block 搬完时发事件 |

**忘了这一步是"事件通路配好了却什么都不发生"的头号原因**：EVSYS 那边全都对，但发生器根本没往外发。顺带说清一件事：**EVSYS 自己没有"总使能"位**，手册 24.6.2.2 的原话是 "The EVSYS is always enabled"——通道的"开关"就是 `CHANNEL.EVGEN` 选没选发生器（复位值 0 = 不接任何发生器），所以本章代码里找不到 `EVSYS.ENABLE` 这样的位。

## 11.4 通道里的路径与边沿：事件的"脉冲语义"

事件在电气上是一个**脉冲**（来了就是来了，不会一直保持）。通道有两个可选动作：**PATH（路径）**决定信号怎么从发生器走到用户，**EDGSEL（边沿检测）**决定认脉冲的哪一条边。

| 路径 | 什么时候用 | 需要通道的通用时钟吗 | 能产生中断 / 状态吗 |
|---|---|---|---|
| Asynchronous（异步） | 发生器与用户同一条 GCLK，要最低延迟 | 不需要 | 不能（`CHSTATUS` 恒为 0） |
| Synchronous（同步） | 发生器与通道同一条 GCLK | 需要 | 能 |
| Resynchronized（重同步） | 发生器与通道时钟不同 | 需要 | 能 |

**手册的 Table 24-2（User Multiplexer Selection）会告诉你这个用户支持哪条路**，这是选路径的第一依据：

| 用户 | 支持的路径 |
|---|---|
| DMAC CH0-3 | **只能用重同步路径** |
| ADC START / ADC SYNC / DAC START / AC COMPx | 只能用异步路径 |
| TCC0-2、TCC3、TC3-7 | 表里写三条都可以 |

DMAC 只支持重同步路径这一点很重要：**只要用户是 DMAC，`PATH` 就写 `RESYNCHRONIZED`，并且这条通道必须配一条通用时钟（GCLK）**，例程一就是这么做的。**边沿检测**只在同步 / 重同步路径下有效，三选一：上升沿（`RISING_EDGE`）、下降沿（`FALLING_EDGE`）、双边沿（`BOTH_EDGES`）；手册 24.6.2.6 有一条容易忽略的限制——**如果发生器给的是脉冲，不能选双边沿**，只能按脉冲的默认电平选上升沿或下降沿。选择依据很简单：**看发生器的事件是"一个脉冲"还是"一个会翻转的电平"**——ADC 的结果就绪是一个脉冲，用上升沿即可；比较器输出是一个会来回翻转的电平，双边沿才有意义。

## 11.5 把硬件概念翻译成寄存器

现在才看寄存器。每个寄存器都应该能回答一个明确的问题：

| 我要做什么 | SAMD21 的配置 |
|---|---|
| 让 ADC 往外发"结果就绪" | `ADC.EVCTRL` 的 `RESRDYEO` |
| 让这个事件走 0 号通道 | `EVSYS.CHANNEL`（`CHANNEL`=0、`EVGEN`=0x42、`PATH`、`EDGSEL`） |
| 让 DMAC CH0 听 0 号通道 | `EVSYS.USER`（`USER`=0x00、`CHANNEL`=0+1） |
| 告诉 DMAC 描述符放哪 | `DMAC.BASEADDR` / `DMAC.WRBADDR` |
| 描述符写"从哪到哪、搬多少" | 描述符里的 `BTCTRL` / `BTCNT` / `SRCADDR` / `DSTADDR` |
| 让 DMAC 每个事件搬一个 beat | `DMAC.CHCTRLB` 的 `EVACT=TRIG`、`EVIE=1`、`TRIGACT=BEAT`、`TRIGSRC=DISABLE` |
| 手动触发一次 | `DMAC.SWTRIGCTRL` 对应位 |
| 看谁在搬、谁在等 | `DMAC.BUSYCH` / `DMAC.PENDCH` / `DMAC.ACTIVE` |
| 知道搬完了 | `DMAC.CHINTFLAG` 的 `TCMPL`（写 1 清） |

**寄存器不是新知识，它只是硬件的控制开关。** 例如例程一要做的事：

```text
我要让"ADC 采完一个样"自动变成"数组里多一个数"
      ↓ 硬件上
ADC 发出一个事件脉冲 → 通道 0 送给 DMAC CH0 → DMAC 读 ADC.RESULT，写进数组
      ↓ SAMD21 上
ADC_EVCTRL.RESRDYEO → EVSYS.USER + EVSYS.CHANNEL → DMAC.CHCTRLB + 描述符
```

**关于代码里的名字**：本书统一用 SAMD21_DFP 3.7.262 的写法——外设指针是 `EVSYS_REGS` / `DMAC_REGS`，寄存器成员带外设前缀，例如 `EVSYS_REGS->EVSYS_CHANNEL`。网上很多教程（以及更老的 Atmel 版 DFP）用 `EVSYS->CHANNEL.reg` 这种写法，两者指的是同一块硬件：

| 本章写法（DFP 3.7.262） | 老写法 / 网上常见 | 说明 |
|---|---|---|
| `EVSYS_REGS->EVSYS_CHANNEL` | `EVSYS->CHANNEL.reg` | 32 位；间接寻址，先写通道号 |
| `EVSYS_REGS->EVSYS_USER` | `EVSYS->USER.reg` | 16 位；必须一次写完 |
| `DMAC_REGS->DMAC_CHCTRLB` | `DMAC->CHCTRLB.reg` | 32 位；作用在 `CHID` 选中的通道上 |
| `dmac_descriptor_registers_t` | （描述符结构体） | 描述符在 **SRAM** 里，不是寄存器 |

另外，这些外设在复位后都**没有被 PAC 写保护**（手册 11.6.2：PAC2 的 `WPCLR` 复位值 0x00800000，只有 AC1 那一位是 1；PAC1 的复位值 0x000002，只有 DSU 是 1），所以代码里不需要先解锁——寄存器描述里写的 "PAC Write-Protection" 说的是"可以被保护"，不是"复位后就锁着"。

## 11.6 EVSYS 寄存器逐个看

EVSYS 一共只有 7 个寄存器，因为它把"12 条通道 / 29 个用户"压缩成了两组**间接寻址**的寄存器。

### CHANNEL（0x04）：回答"这条通道接谁、怎么走"

12 条通道共用这一个 32 位寄存器：**先写 `CHANNEL[3:0]` 选通道号，同一次写入里把配置一起带上**（手册要求一次 32 位写完，分两次写会得到半个配置）。

```text
EVSYS.CHANNEL（一次 32 位写完；其余位保留）
 27:26    25:24    22:16     8        3:0
┌───────┬────────┬─────────┬───────┬──────────┐
│EDGSEL │  PATH  │  EVGEN  │ SWEVT │ CHANNEL  │
└───────┴────────┴─────────┴───────┴──────────┘
 边沿检测  哪条路   选发生器  软件事件  选通道号
```

`SWEVT`（bit8）是个特殊位：往它写 1 等于"我手动替发生器发一个事件"，用于调试；它和 `CHANNEL` 字段必须在同一次写入里给出，读回来永远是 0。`EVGEN` 写 0 表示这条通道不接任何发生器——这就是通道的"关闭"方式。

### USER（0x08）：回答"哪个用户听哪条通道"

29 个用户也共用一组寄存器：低 8 位 `USER` 是用户编号（来自 Table 12-1 的 User 列），bit12:8 的 `CHANNEL` 是**通道号 + 1**。也要一次写完（手册要求 16 位单次写入）。

### CTRL（0x00）：回答"通道的时钟怎么给"

只用到两位：`SWRST`（写 1 复位整个 EVSYS，所有通道配置清零）和 `GCLKREQ`。`GCLKREQ` 值得单独说：置 0（复位默认）时通道工作在 **SleepWalking** 模式——通用时钟平时关着，只有真要传事件时才临时打开，传完就关；置 1 则让通道时钟一直开着。**默认值就是省电的那一个，通常不用动。**

### CHSTATUS（0x0C）/ INTENCLR / INTENSET / INTFLAG（0x10 / 0x14 / 0x18）：回答"事件有没有丢"

`CHSTATUS.CHBUSYn` 表示通道 n 的事件还没被所有用户处理完，`USRRDYn` 表示通道 n 的用户都准备好接下一个事件了，两者**只在同步 / 重同步路径下有效**。`INTFLAG` 里 `OVRn` 是溢出（上一个事件还没处理完，新的又来了），`EVDn` 是检测到一个事件；这两个标志同样只在重同步路径下有效（异步路径读回来恒为 0，手册 24.6.3.1）。调试事件通路时：**把 `OVR` 和 `EVD` 都打开，如果 `EVD` 在涨而 `OVR` 也在涨，说明用户（多半是 DMAC）来不及处理**；如果 `EVD` 一直是 0，问题在发生器侧或通道配置。

## 11.7 DMAC 是什么：12 条通道与四层传输

DMAC（Direct Memory Access Controller）里有两个引擎：**DMA 引擎**（搬数据）和 **CRC 引擎**（算校验），本节讲 DMA 引擎。它有 **12 条独立的通道**，每条通道可以理解成一个只会做一件事的小助手：**"从现在起，把 A 处的东西搬到 B 处，搬 N 次，每次 X 位"**——12 条通道就是 12 个互不干扰的任务槽。

手册给传输定义了四个层次，看懂这张图，后面描述符里的字段就都有出处了：

```text
transaction（一次事务 = 一整条描述符链；单次传输时只有 1 个 block）
 └─ block（一个描述符负责的量，1 ~ 64K 个 beat，搬完置 TCMPL）
     └─ beat（一次总线读写：8 / 16 / 32 位，由 BTCTRL.BEATSIZE 决定）
          burst = n 个 beat，不可被打断；仲裁在 burst 之间发生
```

**beat** 是最小单位，一次总线读或写，选 16 位还是 32 位取决于外设寄存器宽度（ADC 结果是 16 位 → 选 HWORD）；**block** 是一个描述符负责的全部数据，计数写在 `BTCNT` 里，搬完会置 `CHINTFLAG.TCMPL`；**transaction** 是把多个 block 用描述符串起来（**链表**），本章只讲单次传输，链表属于"以后要用再学"的内容。

### 每条通道各自有什么

| 每条通道各自有 | 在哪 |
|---|---|
| 源地址、目的地址、搬多少 | 描述符里的 `SRCADDR` / `DSTADDR` / `BTCNT` |
| 地址增不递增、beat 多大 | 描述符里的 `BTCTRL` |
| 谁来触发 | `CHCTRLB.TRIGSRC` + `TRIGACT` |
| 优先级（4 档）与是否轮转 | `CHCTRLB.LVL` + `CTRL.LVLENx` + `PRICTRL0` |
| 完成 / 出错 / 挂起标志 | `CHINTFLAG`、`CHSTATUS` |

12 条通道共用一个"当前通道"指针：**`CHID` 寄存器**。所有 `CH*` 寄存器（`CHCTRLA`、`CHCTRLB`、`CHINTFLAG`…）都"作用在 `CHID` 选中的那条通道上"，所以写通道寄存器的第一句永远是 `DMAC_REGS->DMAC_CHID = DMAC_CHID_ID(n);`——忘了这句，改的就是上一次选中的通道，这类 bug 很难查。

### 优先级与 CRC

**优先级**分 4 档（`LVL0`~`LVL3`，数字大的先搬），同一档内部默认"通道号小的先"，可以打开轮转（`PRICTRL0.RRLVLENx`）避免高通道号饿死；每一档还要在 `CTRL` 里用 `LVLENx` **使能**，没使能的档位上的请求会被直接忽略（手册 20.8.1）。**CRC 引擎**支持 CRC-16（CRC-CCITT）和 CRC-32（IEEE 802.3），输入可以是**某一条 DMA 通道上流过的数据**，也可以是 CPU 通过 `CRCDATAIN` 逐个写进去的数据，结果在 `CRCCHKSUM` 里，用途是"串口/USB 收到的数据边搬边算校验，CPU 不用再遍历一遍数组"，本章不展开。

## 11.8 描述符：DMAC 的任务单放在 SRAM 里

这是 SAMD21 的 DMAC 最特别的一点：**通道的"任务单"不在寄存器里，而在 SRAM 里**，这块 SRAM 叫**描述符（descriptor）内存区**；DMAC 工作时自己去 SRAM 取描述符、执行，并把进度回写到另一块 SRAM（**回写区 write-back**）。

| 描述符字段 | 名字 | 回答什么 |
|---|---|---|
| `BTCTRL` | Block Transfer Control | beat 多大（`BEATSIZE`）、源/目的地址增不递增（`SRCINC`/`DSTINC`）、块结束后干什么（`BLOCKACT`）、描述符有效吗（`VALID`） |
| `BTCNT` | Block Transfer Count | 搬多少个 beat |
| `SRCADDR` | Source Address | 源地址 |
| `DSTADDR` | Destination Address | 目的地址 |
| `DESCADDR` | Next Descriptor Address | 下一条描述符在哪；写 0 表示"没有下一条"（单次传输） |

描述符的布局是固定的 16 字节（`BTCTRL` 2 字节 + `BTCNT` 2 字节 + 三个 4 字节地址），所以**通道 n 的描述符地址 = 基地址 + n × 0x10**。三条硬性要求：① **描述符区和回写区必须在 SRAM**（DMAC 靠 AHB 去取，Flash 里不行）；② **两个基地址（`BASEADDR` / `WRBADDR`）必须 8 字节对齐**（手册原文：must be 64-bit aligned）；③ **一个区里的描述符按通道号连续排列**，`BASEADDR` 指向通道 0 的描述符，通道 1 紧跟在后面。DFP 头文件已经把布局定义成结构体 `dmac_descriptor_registers_t`（带 `aligned(8)` 属性），所以最省事的写法是直接声明这个类型的数组，而不是自己算偏移。

### 最容易写错的一条：递增时写"最后一个 beat 的地址"

`SRCADDR` / `DSTADDR` 里填的**不是首地址**，而是**最后一个 beat 的地址**：手册 20.10.3 给的关系是 `地址 = 起始地址 + (BTCNT - 1) × BEATSIZE`（地址递增时）；地址不递增时，它同时也是唯一的那个地址。换句话说，**DMAC 从你给的地址开始，边搬边往前减，一直减回起始地址**：

```text
目的 = uint16_t buf[64]，要存 64 个半字
  递增时 DSTADDR 要写 &buf[63]，不是 &buf[0]     ← 写错了会从数组尾部往前写
  不递增（外设寄存器固定）时 SRCADDR 写寄存器地址本身，即 &ADC_REGS->ADC_RESULT
```

如果地址写错，现象是"数据搬了，但落在数组外面的内存里"——数组看着没变，别的地方被改写，非常难查。

## 11.9 触发从哪来：软件、事件、外设专用

一条配置好的通道要"动起来"，必须收到一个**触发**（trigger），来源用 `CHCTRLB.TRIGSRC` 选：

| 触发源 | `TRIGSRC` 写什么 | 适合什么 |
|---|---|---|
| 软件触发 | `DISABLE`（0x00），然后写 `SWTRIGCTRL` 对应位 | 手动搬一块内存；调试 |
| EVSYS 事件 | `DISABLE`（0x00），同时 `EVACT=TRIG`、`EVIE=1` | 外设事件驱动，事件还能广播给别的外设 |
| 外设专用触发 | 表里的编号（SERCOM0 RX=0x01、ADC RESRDY=0x27、DAC EMPTY=0x28…） | 点到点最快的一条路，只有 DMAC 用得到 |

注意前两种的 `TRIGSRC` 都写 `DISABLE`，区别在 `EVACT`/`EVIE`：**"DISABLE" 的意思是"不要外设专用触发"，不是"不能用"**。而 `TRIGACT` 决定"一个触发干多少活"：`BLOCK` 是整个 block（`BTCNT` 个 beat 一口气搬完，适合内存→内存或已知长度的缓冲区）、`BEAT` 是只搬一个 beat、下一个 beat 等下一个触发（适合 ADC 每来一个结果搬一个，见例程一）、`TRANSACTION` 是一整条描述符链。另外 `CHCTRLB.CMD` 是另一回事：它给的是**挂起（SUSPEND）/ 恢复（RESUME）**命令，不是触发。

## 11.10 DMAC 寄存器逐个看

DMAC 的寄存器分两组：**DMAC 自己的**（偏移 0x00~0x38）和**"当前通道"的**（偏移 0x3F~0x4F，受 `CHID` 影响）。

### CTRL（0x00）/ CRCCTRL（0x02）/ QOSCTRL（0x0E）

- `CTRL`：`DMAENABLE`（bit1）是总开关，`LVLEN0`~`LVLEN3`（bit8-11）分别使能 4 个优先级档，`CRCENABLE`（bit2）开 CRC，`SWRST`（bit0）复位整个 DMAC。**它涉及 enable-protection**：`BASEADDR`、`WRBADDR` 只能在 `DMAENABLE=0` 时写，`SWRST` 还要求 `CRCENABLE=0`（手册 20.6.2.1）——这就是配置顺序的由来。
- `CRCCTRL`：`CRCSRC[5:0]` 选输入源（`0x01` = I/O 接口，`0x20`~`0x2B` = DMA 通道 0~11）、`CRCPOLY` 选多项式、`CRCBEATSIZE` 选 I/O 输入时的字节宽度；同样 enable-protected（要在 `CRCENABLE=0` 时写）。
- `QOSCTRL`：三组 2 位——`DQOS`（数据搬运）、`FQOS`（取描述符）、`WRBQOS`（回写描述符），每档从"后台"到"关键延迟"。复位值全 0（后台），普通应用不动它；音频、高速 SPI 这类对延迟敏感的场合才往高档调。

### SWTRIGCTRL（0x10）/ PRICTRL0（0x14）/ BASEADDR（0x34）/ WRBADDR（0x38）

- `SWTRIGCTRL`：低 12 位，一位一条通道。写 1 产生一次软件触发；如果该通道已经 pending（`CHSTATUS.PEND=1`）就先置位等下一次；写 0 会清掉这一位（手册 20.8.8）。所以最干净的写法是 `DMAC_REGS->DMAC_SWTRIGCTRL = (1UL << n);`。
- `PRICTRL0`：`LVLPRIx[3:0]` 记录该档上次被授权的通道号（轮转用），`RRLVLENx` 打开轮转。
- `BASEADDR` / `WRBADDR`：都是 32 位 SRAM 地址，**必须 8 字节对齐**，且只能在 `DMAENABLE=0` 时写。两个区可以相同（省内存），分开的好处是**同一条描述符可以重复使用**：进度写在回写区，不污染原描述符。本章例程用分开的两个区。

### CHID（0x3F）/ CHCTRLA（0x40）/ CHCTRLB（0x44）

`CHID.ID[3:0]` 选择后面 `CH*` 寄存器作用在哪条通道（0~11）；`CHCTRLA.ENABLE` 是通道开关，`CHCTRLA.SWRST` 复位这条通道（要求通道已关闭）；`CHCTRLB` 里有 `TRIGACT`、`TRIGSRC`、`EVACT`、`EVIE`（允许事件输入）、`EVOE`（允许事件输出）、`LVL`（优先级档）、`CMD`（挂起/恢复），**除 `CMD` 和 `LVL` 外，其余位都要求通道处于关闭状态才能写**。一个常被忽略的细节：`EVIE`/`EVOE`/`EVACT` **只在通道 0~3 上有效**（手册 20.2 Features：4 个事件输入、4 个事件输出，对应最低的 4 条通道），所以想让 DMAC 被事件触发，**通道号必须选 0~3**。

### 触发之后看哪里：BUSYCH / PENDCH / INTSTATUS / INTPEND / ACTIVE

| 寄存器 | 偏移 | 回答什么 |
|---|---|---|
| `INTSTATUS` | 0x24 | 位图：哪些通道有挂起的中断 |
| `BUSYCH` | 0x28 | 位图：哪些通道正在搬（第一个 burst 开始置 1，传输结束清 0） |
| `PENDCH` | 0x2C | 位图：哪些通道收到了触发但还没轮到它 |
| `ACTIVE` | 0x30 | 当前活动通道号 `ID`、`ABUSY`、剩余 `BTCNT`、哪些优先级档在执行 |
| `INTPEND` | 0x20 | **编号最小的**有中断的通道号 + 它的 `TCMPL`/`TERR`/`SUSP`/`FERR`/`BUSY`/`PEND`，省得自己遍历 12 位 |

调试 DMA 的通用手法：**触发之后先看 `PENDCH` 有没有置位**（说明触发到了 DMAC），**再看 `BUSYCH`**（说明真的开始搬了），**最后看 `CHINTFLAG.TCMPL`**（说明搬完了）——卡在哪一步，问题就在哪一段。每通道还有三个中断源：`TCMPL`（block 搬完）、`TERR`（总线错误）、`SUSP`（通道被挂起），写 1 到 `CHINTFLAG`（0x4E）对应位清除；`CHINTENSET`/`CHINTENCLR`（0x4D/0x4C）开关它们；`CHSTATUS`（0x4F）里有 `PEND`（有触发在等）、`BUSY`（在搬）、`FERR`（取到无效描述符）。

## 11.11 一次 DMA 传输的配置顺序

顺序不是习惯，是硬件要求（手册 20.6.2.1 把每一步都写明了），按这个顺序走就不会踩 enable-protection：

| 步骤 | 做什么 | 为什么必须在这个位置 |
|---|---|---|
| 1 | 打开 DMAC 的总线时钟 | 桥 B 的 AHB/APB 时钟复位后就是开的（Table 12-1），本章不需要写 |
| 2 | 写 `BASEADDR` / `WRBADDR`，写 `CTRL.LVLENx` | 这两个地址寄存器要求 `DMAENABLE=0` |
| 3 | 写 `CTRL.DMAENABLE=1` | 从这一步起，地址寄存器就锁上了 |
| 4 | 填描述符（`BTCTRL`/`BTCNT`/`SRCADDR`/`DSTADDR`/`DESCADDR`），最后置 `BTCTRL.VALID=1` | 描述符无效时通道会被挂起并置 `FERR` |
| 5 | `CHID` 选通道 | 后面所有 `CH*` 寄存器都跟着它走 |
| 6 | 写 `CHCTRLB`（`TRIGACT`/`TRIGSRC`/`EVACT`/`EVIE`/`LVL`） | 这些位要求通道处于关闭状态 |
| 7 | 清一次 `CHINTFLAG`，再写 `CHCTRLA.ENABLE=1` | 清旧标志，免得把上次的完成当成这次的 |
| 8 | 触发：写 `SWTRIGCTRL`，或等 EVSYS 送来事件 | 通道必须先使能，触发才有意义 |
| 9 | 等 `CHINTFLAG.TCMPL`，或等 `BUSYCH` 对应位清 0 | 传输是异步的，不等就改描述符会出错 |
| 10 | 清 `TCMPL`；要复用就重填描述符 | block 搬完硬件会把 `VALID` 清 0、通道自动关闭 |

**第 10 步是最容易忘的**：一次传输结束后，描述符的 `VALID` 位被硬件清 0 了（手册 20.10.1），通道也自动关闭了。想再搬一次，必须**重新填一遍描述符里的地址和计数、重新置 `VALID`**，再重新 `ENABLE`。

## 11.12 例程一：ADC 自由运行 + 事件触发 + DMA 搬 64 个采样值

```text
PA03/AIN1 ─▶ ADC（自由运行）─ 结果写进 ADC.RESULT，并发出 RESRDY 事件脉冲
              │ ADC.EVCTRL.RESRDYEO=1
              ▼
   EVSYS 通道 0（发生器 0x42，重同步，上升沿）→ EVSYS.USER：DMAC CH0 挂在通道 0
              ▼
   每个事件读一次 ADC.RESULT 写 adc_buf[]；搬完 64 个 → TCMPL 置位、通道自动关闭
```

这里有一个漂亮的副作用：**DMAC 读 `ADC.RESULT` 这个动作本身，就清掉了 ADC 的 DMA 请求**（手册 33.6.11：请求在结果可用时置位、在 `RESULT` 被读走时清除），所以每来一个新采样都能重新触发一次，不需要 CPU 插手。硬件上只需要：**PA03 接一个 0~VDDANA 之间的模拟电压**（VDDANA = 3.3 V 时参考选 1/2 VDDANA，满量程约 1.65 V）；PA03 = ADC AIN1（手册 Table 7-1），要关掉该脚的数字输入缓冲。ADC 本体用"自由运行（`CTRLB.FREERUN=1`）+ 12 位 + 1/2 VDDANA 参考"，其余细节见第 10 章；搬运用半字（16 位），源地址固定、目的地址递增，`BTCNT=64`。

### 完整代码

```c
#include "sam.h"
#include <stdint.h>

#define SAMPLE_COUNT 64

/* DMA 的目的缓冲区：64 个 12 位采样（右对齐存在 16 位里） */
static uint16_t adc_buf[SAMPLE_COUNT];

/* 描述符区与回写区：必须在 SRAM，且 8 字节对齐。
   DFP 里的 dmac_descriptor_registers_t 已经带 aligned(8) 属性 */
static dmac_descriptor_registers_t desc_section[1];
static dmac_descriptor_registers_t wb_section[1];

/* ① ADC 本体：自由运行 + 12 位 + 1/2 VDDANA 参考（详见第 10 章） */
static void adc_init(void)
{
    /* ADC 挂在桥 C，APB 时钟复位后是关的（第 2 章） */
    PM_REGS->PM_APBCMASK |= PM_APBCMASK_ADC_Msk;

    /* ADC 需要一个通用时钟：这里用 GCLK0（第 5 章的完整用法） */
    GCLK_REGS->GCLK_CLKCTRL = GCLK_CLKCTRL_ID_ADC
                            | GCLK_CLKCTRL_GEN_GCLK0
                            | GCLK_CLKCTRL_CLKEN_Msk;
    while (GCLK_REGS->GCLK_STATUS & GCLK_STATUS_SYNCBUSY_Msk) { }

    /* PA03 作模拟输入：关掉数字输入缓冲（避免模拟脚上的额外电流），方向设为输入 */
    PORT_REGS->GROUP[0].PORT_PINCFG[3] = 0;
    PORT_REGS->GROUP[0].PORT_DIRCLR  = (1UL << 3);

    /* 软件复位 ADC，确保从已知状态开始 */
    ADC_REGS->ADC_CTRLA = ADC_CTRLA_SWRST_Msk;

    /* 参考电压 = 1/2 VDDANA；分辨率 12 位；自由运行；输入 AIN1 对地；
       AVGCTRL / SAMPCTRL 保持复位值：不累加、最短采样时间 */
    ADC_REGS->ADC_REFCTRL   = ADC_REFCTRL_REFSEL_INTVCC1;
    ADC_REGS->ADC_CTRLB     = ADC_CTRLB_RESSEL_12BIT | ADC_CTRLB_FREERUN_Msk;
    ADC_REGS->ADC_INPUTCTRL = ADC_INPUTCTRL_MUXPOS_PIN1     /* AIN1 = PA03 */
                            | ADC_INPUTCTRL_MUXNEG_GND;

    /* 打开事件输出：这才是"发生器侧开关"，不置它后面全都不动 */
    ADC_REGS->ADC_EVCTRL = ADC_EVCTRL_RESRDYEO_Msk;

    /* 使能 ADC；因为 FREERUN=1，转换会一个接一个自动进行 */
    ADC_REGS->ADC_CTRLA = ADC_CTRLA_ENABLE_Msk;
    while (ADC_REGS->ADC_STATUS & ADC_STATUS_SYNCBUSY_Msk) { }
}

/* ② EVSYS：ADC RESRDY → 通道 0 → DMAC CH0（必须先 USER，后 CHANNEL） */
static void evsys_init(void)
{
    PM_REGS->PM_APBCMASK |= PM_APBCMASK_EVSYS_Msk;

    /* 通道 0 走重同步路径，需要这条通道自己的通用时钟（第 5 章） */
    GCLK_REGS->GCLK_CLKCTRL = GCLK_CLKCTRL_ID_EVSYS_0
                            | GCLK_CLKCTRL_GEN_GCLK0
                            | GCLK_CLKCTRL_CLKEN_Msk;
    while (GCLK_REGS->GCLK_STATUS & GCLK_STATUS_SYNCBUSY_Msk) { }

    /* 先挂用户：USER = 0x00 是 DMAC CH0；CHANNEL 字段写"通道号 + 1" */
    EVSYS_REGS->EVSYS_USER = EVSYS_USER_USER(0x00) | EVSYS_USER_CHANNEL(0 + 1);

    /* 再配通道 0：发生器 0x42 = ADC RESRDY；重同步；上升沿。一次 32 位写完 */
    EVSYS_REGS->EVSYS_CHANNEL = EVSYS_CHANNEL_CHANNEL(0)
                              | EVSYS_CHANNEL_EVGEN(0x42)
                              | EVSYS_CHANNEL_PATH_RESYNCHRONIZED
                              | EVSYS_CHANNEL_EDGSEL_RISING_EDGE;
}

/* ③ 描述符：真正的"任务单" */
static void dmac_descriptor_init(void)
{
    /* 半字搬运；源地址不递增（始终读 ADC.RESULT）；目的递增；块结束产生中断 */
    desc_section[0].DMAC_BTCTRL = DMAC_BTCTRL_VALID_Msk
                                | DMAC_BTCTRL_BEATSIZE_HWORD
                                | DMAC_BTCTRL_DSTINC_Msk
                                | DMAC_BTCTRL_BLOCKACT_INT;

    desc_section[0].DMAC_BTCNT = DMAC_BTCNT_BTCNT(SAMPLE_COUNT);

    /* 源地址固定：写寄存器地址本身 */
    desc_section[0].DMAC_SRCADDR = (uint32_t)&ADC_REGS->ADC_RESULT;

    /* 目的递增：写"最后一个 beat"的地址，不是首地址（11.8 节） */
    desc_section[0].DMAC_DSTADDR = (uint32_t)&adc_buf[SAMPLE_COUNT - 1];

    /* 单次传输：没有下一条描述符 */
    desc_section[0].DMAC_DESCADDR = 0;
}

/* ④ DMAC：DMAC 级配置 → 通道配置 → 使能 */
static void dmac_init(void)
{
    /* 两个区的地址必须在 DMAC 关闭时写 */
    DMAC_REGS->DMAC_BASEADDR = (uint32_t)desc_section;
    DMAC_REGS->DMAC_WRBADDR  = (uint32_t)wb_section;

    /* 打开 DMAC，并启用优先级档 0（本例只用到这一档） */
    DMAC_REGS->DMAC_CTRL = DMAC_CTRL_DMAENABLE_Msk | DMAC_CTRL_LVLEN0_Msk;

    /* 选通道 0：之后的 CH* 寄存器都作用在它身上 */
    DMAC_REGS->DMAC_CHID = DMAC_CHID_ID(0);

    /* TRIGSRC=DISABLE：不用外设专用触发；
       EVACT=TRIG + EVIE=1：事件到来就执行传输；
       TRIGACT=BEAT：一个事件只搬一个半字 */
    DMAC_REGS->DMAC_CHCTRLB = DMAC_CHCTRLB_TRIGSRC_DISABLE
                            | DMAC_CHCTRLB_TRIGACT_BEAT
                            | DMAC_CHCTRLB_EVACT_TRIG
                            | DMAC_CHCTRLB_EVIE_Msk
                            | DMAC_CHCTRLB_LVL_LVL0;

    /* 清掉可能残留的完成标志，再使能通道 */
    DMAC_REGS->DMAC_CHINTFLAG = DMAC_CHINTFLAG_TCMPL_Msk;
    DMAC_REGS->DMAC_CHCTRLA   = DMAC_CHCTRLA_ENABLE_Msk;
}

int main(void)
{
    adc_init();
    evsys_init();
    dmac_descriptor_init();
    dmac_init();

    /* 启动第一次转换；之后 ADC 自己连续转，每转完一个就触发一次 DMAC */
    ADC_REGS->ADC_SWTRIG = ADC_SWTRIG_START_Msk;

    /* 64 个采样搬完之前 CPU 可以做别的事，这里就等这一批结束 */
    while ((DMAC_REGS->DMAC_CHINTFLAG & DMAC_CHINTFLAG_TCMPL_Msk) == 0) { }

    /* 写 1 清完成标志；此时 adc_buf[0..63] 里是 64 个采样值 */
    DMAC_REGS->DMAC_CHINTFLAG = DMAC_CHINTFLAG_TCMPL_Msk;

    while (1) { }
}
```

按 11.10 节的手法分三步确认它在工作，卡在哪一步就查哪一段：

| 看什么 | 期望 | 不对时说明什么 |
|---|---|---|
| `DMAC_PENDCH` 的 bit0 | 采样开始后不断有置位 | 事件没到 DMAC：查 `ADC_EVCTRL.RESRDYEO`、`EVSYS_USER` 的通道号是不是写成了 0、`EVSYS_CHANNEL.EVGEN` |
| `DMAC_BUSYCH` 的 bit0 | 传输期间为 1 | 触发了但没搬：查描述符 `VALID`、`CHCTRLA.ENABLE`、`CHCTRLB` 是不是在通道使能之后才写的 |
| `CHINTFLAG.TCMPL` | 64 个采样后置 1 | 只搬一两个就停：查 `TRIGACT` 是不是 `BEAT`、`BTCNT` 是不是 64 |
| `EVSYS_INTFLAG.OVR0` | 一直为 0 | 用户在丢事件：DMAC 来不及处理（本例不会，DMAC 比 ADC 快得多） |

## 11.13 例程二：软件触发，把一块内存搬到另一块

这个例子把 EVSYS 完全摘掉，只用 DMAC，用来验证"描述符 + 软件触发"这条最基本的链路；用通道 1（避开通道 0，方便和例程一同时存在）。

```c
#include "sam.h"
#include <stdint.h>

#define WORD_COUNT 8

static uint32_t src_buf[WORD_COUNT] = { 1, 2, 3, 4, 5, 6, 7, 8 };
static uint32_t dst_buf[WORD_COUNT];

static dmac_descriptor_registers_t desc_section[1];
static dmac_descriptor_registers_t wb_section[1];

int main(void)
{
    /* ① 描述符：32 位 beat；源和目的都递增；块结束不产生中断、通道自动关闭 */
    desc_section[0].DMAC_BTCTRL = DMAC_BTCTRL_VALID_Msk
                                | DMAC_BTCTRL_BEATSIZE_WORD
                                | DMAC_BTCTRL_SRCINC_Msk
                                | DMAC_BTCTRL_DSTINC_Msk;

    desc_section[0].DMAC_BTCNT = DMAC_BTCNT_BTCNT(WORD_COUNT);

    /* 两个地址都递增 → 都写"最后一个 beat"的地址 */
    desc_section[0].DMAC_SRCADDR = (uint32_t)&src_buf[WORD_COUNT - 1];
    desc_section[0].DMAC_DSTADDR = (uint32_t)&dst_buf[WORD_COUNT - 1];

    desc_section[0].DMAC_DESCADDR = 0;              /* 单次传输 */

    /* ② DMAC 级配置（必须在 DMAENABLE=1 之前） */
    DMAC_REGS->DMAC_BASEADDR = (uint32_t)desc_section;
    DMAC_REGS->DMAC_WRBADDR  = (uint32_t)wb_section;
    DMAC_REGS->DMAC_CTRL     = DMAC_CTRL_DMAENABLE_Msk | DMAC_CTRL_LVLEN0_Msk;

    /* ③ 选通道 1，配成"软件触发、一个触发搬完整块" */
    DMAC_REGS->DMAC_CHID = DMAC_CHID_ID(1);
    DMAC_REGS->DMAC_CHCTRLB = DMAC_CHCTRLB_TRIGSRC_DISABLE
                            | DMAC_CHCTRLB_TRIGACT_BLOCK
                            | DMAC_CHCTRLB_EVACT_NOACT
                            | DMAC_CHCTRLB_LVL_LVL0;

    DMAC_REGS->DMAC_CHINTFLAG = DMAC_CHINTFLAG_TCMPL_Msk;   /* 清旧标志 */
    DMAC_REGS->DMAC_CHCTRLA   = DMAC_CHCTRLA_ENABLE_Msk;    /* 打开通道 */

    /* ④ 软件触发：往通道 1 那一位写 1 */
    DMAC_REGS->DMAC_SWTRIGCTRL = (1UL << 1);

    /* ⑤ 等这一块搬完 */
    while ((DMAC_REGS->DMAC_CHINTFLAG & DMAC_CHINTFLAG_TCMPL_Msk) == 0) { }
    DMAC_REGS->DMAC_CHINTFLAG = DMAC_CHINTFLAG_TCMPL_Msk;

    /* 此时 dst_buf[] 与 src_buf[] 内容相同，通道已自动关闭、VALID 已被清 0 */

    while (1) { }
}
```

和例程一的差别只有三处：**没有 EVSYS**、`TRIGACT` 用 `BLOCK`（一个触发搬完整块）、触发靠 `SWTRIGCTRL`。想再搬一次，就要回到 11.11 节第 10 步：重填描述符（地址、`BTCNT`、`VALID`）再 `ENABLE`。

## 11.14 初始化清单

以后遇到"要用事件"或"要用 DMA"，按这个顺序想，就不会漏：

**只用事件（外设触发外设）**：发生器侧的 `EVCTRL` 打开 → `EVSYS.USER` 挂用户（通道号 + 1）→ `EVSYS.CHANNEL` 选发生器 / 路径 / 边沿 → 需要的话给这条通道配 GCLK。

**用 DMA**：开 DMAC 时钟（桥 B 默认已开）→ `BASEADDR`/`WRBADDR`（8 字节对齐）→ `CTRL.DMAENABLE` + `LVLENx` → 填描述符并置 `VALID` → `CHID` 选通道 → `CHCTRLB`（`TRIGACT`/`TRIGSRC`/`EVACT`/`EVIE`）→ 清 `CHINTFLAG` → `CHCTRLA.ENABLE` → 触发 → 等 `TCMPL`。

**事件 + DMA 一起用**：先按"只用事件"配好发生器和通道（用户是 DMAC），再按"用 DMA"配好 DMAC；`CHCTRLB` 里 `EVACT=TRIG`、`EVIE=1`、`TRIGSRC=DISABLE`；触发动作按需要选 `BEAT`（每个事件搬一个）或 `BLOCK`（每个事件搬一块）。

**运行中不要动的东西**：`BASEADDR`/`WRBADDR`（要求 DMAC 关闭）、`CHCTRLB`（要求通道关闭）、描述符（要求通道已经搬完或已挂起）——要改就先 `CHCTRLA.ENABLE=0` 或发 `CMD.SUSPEND`，改完再重开。

## 11.15 思维模型

```text
事件与传输
├─ EVSYS 只传"发生了"：发生器 --通道(选发生器/路径/边沿)--> 用户（发生器侧开关在各外设的 EVCTRL）
└─ DMAC 只搬数据
    ├─ 谁触发：SWTRIGCTRL / EVSYS 事件(EVIE+EVACT) / 外设专用 TRIGSRC
    ├─ 搬什么：描述符（在 SRAM 里，BASEADDR 8 字节对齐，通道 n 在 +n×0x10）
    └─ 搬多少：BTCTRL(beat 大小/递增) + BTCNT + SRCADDR/DSTADDR(递增时写末地址)
```

一句话总结：**EVSYS 决定"什么时候"，DMAC 决定"搬什么"，描述符是它们之间的合同。**

## 11.16 常见问题

| 现象 | 原因 | 检查什么 |
|---|---|---|
| DMA 完全不启动 | 通道没使能，或改错了通道 | `CHCTRLA.ENABLE`、`CHID` 是不是选对了通道 |
| 用事件触发时完全没反应 | 发生器侧没打开事件输出 | 发生器自己的 `EVCTRL`（ADC 的 `RESRDYEO`） |
| 事件配好了但 DMAC 不动 | `EVIE` 没置 1、`EVACT` 没选 `TRIG`，或 `USER.CHANNEL` 写成了通道号本身 | `CHCTRLB.EVIE`/`EVACT`；想接通道 0 要写 **1**，写 0 等于不接 |
| 事件配好了但 DMAC 不动（二） | 用户挂到了通道 4~11 | `EVIE`/`EVACT` 只在通道 0~3 有效 |
| 通道一使能就挂起，`FERR=1` | 描述符 `VALID=0`，或 `DESCADDR` 指向了野地址 | `BTCTRL.VALID`、`DESCADDR` |
| 数据搬了但落在别处 | 递增模式下地址写成了首地址 | `SRCADDR`/`DSTADDR` 应为"起始 + (N−1)×beat" |
| 传输报 `TERR` | 访问了未对齐地址，或外设时钟没开 | 半字数组是否 2 字节对齐、字数组是否 4 字节对齐；源/目的外设的时钟 |
| 改了描述符却不生效 | 通道正在搬，或 `VALID` 已被硬件清 0 | 等 `TCMPL`/`BUSYCH` 清零；重填后重新置 `VALID` |
| 通道寄存器写不进去 | 撞上 enable-protection | `BASEADDR`/`WRBADDR` 要求 `DMAENABLE=0`；`CHCTRLB` 要求通道关闭 |
| 第二次传输不搬 | `VALID` 已被清 0、通道已自动关闭 | 重填描述符并重新 `ENABLE` |
| 高通道号永远轮不到 | 同优先级档内按通道号静态仲裁 | 打开 `PRICTRL0.RRLVLENx` 轮转 |
| 某个优先级档的通道不动 | 那一档没在 `CTRL` 里使能 | `CTRL.LVLEN0`~`LVLEN3` |

## 11.17 寄存器速查

**EVSYS**（基址 `0x4200_0400`，桥 C）

| 寄存器 | 偏移 | 作用 |
|---|---|---|
| `CTRL` | 0x00 | `SWRST`（复位）、`GCLKREQ`（通道时钟常开 / 按需） |
| `CHANNEL` | 0x04 | 32 位一次写完：`CHANNEL[3:0]` 通道号、`SWEVT` bit8、`EVGEN[6:0]` bit22:16、`PATH[1:0]` bit25:24、`EDGSEL[1:0]` bit27:26 |
| `USER` | 0x08 | 16 位一次写完：`USER[7:0]` 用户编号、`CHANNEL[4:0]` bit12:8 = 通道号 + 1 |
| `CHSTATUS` | 0x0C | `USRRDYn` bit7:0、`CHBUSYn` bit15:8（仅同步/重同步路径有效） |
| `INTENCLR` / `INTENSET` / `INTFLAG` | 0x10 / 0x14 / 0x18 | `OVRn` 低 12 位、`EVDn` 高 12 位，写 1 清 |

**DMAC**（基址 `0x4100_4800`，桥 B）

| 寄存器 | 偏移 | 作用 |
|---|---|---|
| `CTRL` | 0x00 | `SWRST`、`DMAENABLE`、`CRCENABLE`、`LVLEN0-3`（bit8-11） |
| `CRCCTRL` | 0x02 | `CRCBEATSIZE`、`CRCPOLY`、`CRCSRC`（0x01=I/O，0x20+n=通道 n） |
| `CRCSTATUS` | 0x0C | `CRCBUSY`、`CRCZERO` |
| `QOSCTRL` | 0x0E | `WRBQOS`、`FQOS`、`DQOS` 各 2 位 |
| `SWTRIGCTRL` | 0x10 | 低 12 位，一位一条通道 |
| `PRICTRL0` | 0x14 | `LVLPRIx[3:0]`、`RRLVLENx` |
| `INTPEND` | 0x20 | `ID[3:0]`、`TERR`/`TCMPL`/`SUSP`/`FERR`/`BUSY`/`PEND` |
| `INTSTATUS` / `BUSYCH` / `PENDCH` | 0x24 / 0x28 / 0x2C | 12 位位图 |
| `ACTIVE` | 0x30 | `LVLEXx`、`ABUSY`、`ID[4:0]`、`BTCNT[15:0]` |
| `BASEADDR` / `WRBADDR` | 0x34 / 0x38 | 描述符区 / 回写区地址（8 字节对齐，`DMAENABLE=0` 时写） |
| `CHID` | 0x3F | `ID[3:0]` 选择当前通道 |
| `CHCTRLA` | 0x40 | `SWRST`、`ENABLE` |
| `CHCTRLB` | 0x44 | `EVACT[2:0]`、`EVIE`、`EVOE`、`LVL[1:0]`、`TRIGSRC[5:0]` bit13:8、`TRIGACT[1:0]` bit23:22、`CMD[1:0]` bit25:24 |
| `CHINTENCLR` / `CHINTENSET` / `CHINTFLAG` / `CHSTATUS` | 0x4C / 0x4D / 0x4E / 0x4F | `TERR`/`TCMPL`/`SUSP`；`CHSTATUS` 里是 `PEND`/`BUSY`/`FERR` |

**描述符**（在 SRAM 里，不是外设寄存器；地址 = `BASEADDR/WRBADDR` + 通道号 × 0x10）

| 字段 | 偏移 | 作用 |
|---|---|---|
| `BTCTRL` | 0x00 | `VALID`、`EVOSEL[1:0]`、`BLOCKACT[1:0]`、`BEATSIZE[1:0]`、`SRCINC`、`DSTINC`、`STEPSEL`、`STEPSIZE[2:0]` |
| `BTCNT` | 0x02 | 本块的 beat 数（1~65535） |
| `SRCADDR` | 0x04 | 源地址：递增时写最后一个 beat 的地址 |
| `DSTADDR` | 0x08 | 目的地址：同上 |
| `DESCADDR` | 0x0C | 下一条描述符地址，0 = 结束 |

## 11.18 最重要的几句话

1. 事件只传"发生了"，不传数据；数据要么由 DMAC 搬，要么由外设在事件驱动下自己动。
2. EVSYS 是 12 条连线：一条通道接一个发生器，可以广播给多个用户；用户自己选听哪条通道。
3. 接通一条事件路径要动三处：发生器侧的 `EVCTRL`、`EVSYS.USER`（先）、`EVSYS.CHANNEL`（后）；`USER.CHANNEL` 里写的是**通道号 + 1**，写 0 等于不接。
4. 用户是 DMAC 时，通道路径只能用**重同步**，并且要给它配一条 GCLK；`EVIE`/`EVACT` 只在通道 0~3 有效。
5. DMAC 的任务单（描述符）在 **SRAM** 里，`BASEADDR`/`WRBADDR` 必须 8 字节对齐，通道 n 的描述符在 `+n×0x10`。
6. 地址递增时，`SRCADDR`/`DSTADDR` 写的是**最后一个 beat 的地址**，不是首地址。
7. 顺序是硬件要求：`BASEADDR`/`WRBADDR` 在 `DMAENABLE=0` 时写，`CHCTRLB` 在通道关闭时写，描述符先于通道使能。
8. 一次传输结束后 `VALID` 被清 0、通道自动关闭；要再搬一次必须重填描述符。

## 11.19 自测

1. 事件通道传不传数据？如果 ADC 采到的数值不在事件里，它是怎么进到数组里的？
2. 想把 DMAC CH2 挂到**通道 3** 上，`EVSYS.USER` 里两个字段分别写什么？为什么手册要求先配 USER 再配 CHANNEL？
3. 为什么"用户是 DMAC"时通道路径只能选重同步？这条路径对时钟有什么额外要求？
4. `BASEADDR`/`WRBADDR` 为什么必须 8 字节对齐？描述符能不能放在 Flash 里，为什么？
5. 把 64 个半字从 `ADC.RESULT` 搬到 `buf[64]`，`BTCNT`、`BEATSIZE`、`SRCADDR`、`DSTADDR` 各写什么？`DSTADDR` 写 `buf` 首地址会怎样？
6. 下面的顺序哪里错了，会导致什么现象？

```c
DMAC_REGS->DMAC_CHID = DMAC_CHID_ID(0);
DMAC_REGS->DMAC_CHCTRLA = DMAC_CHCTRLA_ENABLE_Msk;
DMAC_REGS->DMAC_CHCTRLB = DMAC_CHCTRLB_TRIGACT_BEAT | DMAC_CHCTRLB_EVIE_Msk;
DMAC_REGS->DMAC_BASEADDR = (uint32_t)desc_section;
```

## 附录：以后查手册怎么找 EVSYS 和 DMAC

手册先记章节号：**第 24 章 EVSYS**、**第 20 章 DMAC**、总表是**第 12 章 Table 12-1**；时钟在**第 15 章 GCLK**、**第 16 章 PM**；ADC 的事件与 DMA 请求在**第 33 章**（33.6.11 DMA Operation、33.6.13 Events）。查的时候编号一律从表里抄、不要背：

```text
事件：① Table 12-1 找发生器和用户 → 两列的数分别填 EVGEN 和 EVSYS.USER.USER
      ② 用户支持哪条 PATH：第 24.8.3 的 Table 24-2 最后一列
      ③ 发生器侧开关：跳到那个外设章节找 EVCTRL（如 ADC 的 33.8.9）
      ④ 通道位定义与 EVGEN 表：第 24.8.2 CHANNEL
DMA： ① 第 20.6.2.1 Initialization（配置顺序与 enable-protection 的权威清单）
      ② 第 20.8 寄存器描述；③ 第 20.10 描述符字段（SRAM 里）；④ 第 20.8.19 TRIGSRC 表
```

代码里对不上的名字，直接查 DFP 头文件：`component/evsys.h`、`component/dmac.h`（描述符结构体在 `dmac.h` 末尾）。宏只有两种形式——`_Msk` 是掩码，`_Val` 是**未移位的枚举值**；多位字段要用带移位的宏（如 `EVSYS_USER_CHANNEL(1)`）或 `_Val << _Pos`，直接写 `_Val` 会落到错误的位上。
