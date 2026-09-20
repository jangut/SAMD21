# 第 7 章 SERCOM（一）I2C：两根线上的六个外设

> **芯片**：ATSAMD21G18A（Cortex-M0+ @48 MHz，256 KB Flash，32 KB SRAM，48 脚 TQFP/QFN）
> **本章目标**：搞清 SERCOM 为什么是"三合一"，I2C 为什么必须开漏加外接上拉，主机的每一次总线动作对应哪个寄存器和哪个状态位；最后写出"先写寄存器指针、再读回两个字节"的完整例程。
> **学习顺序**：SERCOM 是什么 → 为什么这样设计 → I2C 的电气原理 → 一次传输的全过程 → 哪些引脚能做 → 怎么配（顺序） → 寄存器 → 例程 → 清单 → 模型 → 自测。
> **本章不写**：SPI 和 USART（第 8 章）；中断方式（第 12 章）；DMA（第 11 章）。

## 7.1 为什么一颗芯片要 6 个"可配置串口"

### 7.1.1 三种串行协议，各自向芯片要什么

先用一张表把三种协议摆平。它们在电气层的差别，决定了硬件要怎么设计：

| 协议 | 线数 | 时钟 | 电气方式 | 谁是主人 | 典型用途 |
|---|---|---|---|---|---|
| USART | 2（TX/RX） | 异步，靠波特率约定 | 推挽 | 点对点，无主从 | 串口终端、GPS、蓝牙模块 |
| SPI | 4（SCK/MOSI/MISO/SS） | 同步，主机给 | 推挽 | 一主多从，SS 片选 | Flash、屏幕、SD 卡 |
| I2C | 2（SCL/SDA） | 同步，主机给 | **开漏 + 上拉** | 一主多从，靠地址寻址 | 传感器、EEPROM、RTC |

三条结论直接从这里出来：**USART 和 SPI 的输出是推挽的**（第 4 章那个上管加下管的输出级），**I2C 不是**；USART 不需要时钟线，**SPI 和 I2C 需要**；USART/SPI 的"选谁"靠独立的片选线或独立连线，I2C 的"选谁"靠把地址写进数据流。

### 7.1.2 STM32 是一个协议一套外设，SAMD21 是一个引擎换身份

你在 STM32 上习惯了这种对应：`USART1` 永远是 USART，`SPI1` 永远是 SPI，`I2C1` 永远是 I2C，用不到的协议就是一堆闲置的硅片面积。SAMD21 反过来：**SERCOM**（SERial COMmunication interface，串行通信接口）有最多 6 个实例，每一个都能被配置成 USART、SPI 或 I2C 中的一种。手册 25.1 节的原话是：一旦某个 SERCOM 被配置并使能，**它的全部资源都归所选的那种模式**。选择身份的动作只有一个，就是写 `CTRLA.MODE[2:0]`（手册 Table 25-1）：`0x0` USART 外部时钟、`0x1` USART 内部时钟、`0x2` SPI 从机、`0x3` SPI 主机、`0x4` I2C 从机、**`0x5` I2C 主机（本章）**，`0x6`/`0x7` 保留。一个容易踩的坑：**MODE 只有在 SERCOM 关闭（`CTRLA.ENABLE = 0`）时才能改**，使能之后再写 MODE，硬件直接丢掉，不报错。

### 7.1.3 这样设计换来了什么，代价是什么

| 换来 | 代价 |
|---|---|
| 板上需要什么协议，就把它分给一个 SERCOM；不需要的协议不占外设 | 同一个 SERCOM 不能"半 I2C 半 SPI"，资源是独占的 |
| 6 个实例，最多同时 6 路串口，或 5 路串口加 1 路 I2C | 引脚不能随便挑：某个 SERCOM 只能用布线连到它的那几个脚（7.6 节） |
| 三种协议的寄存器布局尽量对齐（`CTRLA` 都在 0x00，`DATA` 都在 0x28） | 名字相同、含义不同：I2C 从机的 `ADDR` 是"我的地址"，主机的 `ADDR` 是"我要找谁" |

最后一行是本章最该记住的一句话：**寄存器不是新增的知识，只是硬件的控制开关；同一块硬件换身份，同一个偏移上的寄存器就换含义。**

## 7.2 SERCOM 的内部骨架

### 7.2.1 引擎：缓冲加移位寄存器

不管哪种模式，SERCOM 里面都是同一套东西（手册 25.6.1、Figure 25-2）：

| 部件 | 作用 | 三种协议里都存在的理由 |
|---|---|---|
| 发送缓冲（1 级） | 软件写好、还没开始移位的那个字节 | 软件不必等移位结束 |
| 接收缓冲（I2C 1 级，USART/SPI 2 级） | 已经收完、等软件取走的字节 | 软件慢一点也不会丢 |
| 移位寄存器 | 一位一位地进出引脚 | 串行通信的本体 |
| 波特率发生器 | 产生位时钟 | SPI/I2C 的主机、USART 的收发都要 |
| 地址匹配逻辑 | 判断总线上这个地址是不是给我的 | I2C 从机、SPI 从机 |

```text
        ┌──────────── SERCOM ────────────┐
软件 ──▶│ DATA(写) ─▶ 发送缓冲 ─▶ 移位寄存器 │──▶ PAD[0..3] ──▶ 引脚
软件 ◀──│ DATA(读) ◀─ 接收缓冲 ◀─ 移位寄存器 │◀── PAD[0..3] ◀── 引脚
        │ BAUD ─▶ 波特率发生器   地址匹配   │
        └─────────────────────────────────┘
```

注意最右端：**SERCOM 不直接连引脚，它连的是 4 个内部焊盘 PAD[3:0]**，再由 PORT 的复用开关把 PAD 接到具体引脚上。这就是 7.6 节"为什么只有部分引脚能做 I2C"的全部原因。

### 7.2.2 同一块硬件，四套寄存器视图

手册把 I2C 拆成两处讲（28.7–28.8 从机，28.9–28.10 主机），各给一张 Register Summary。把它们和 SPI、USART 的 Summary 并排看，偏移几乎完全重合：

| 偏移 | I2C 主机 | I2C 从机 | SPI | USART |
|---|---|---|---|---|
| 0x00 / 0x04 | `CTRLA`（MODE 选身份） / `CTRLB` | `CTRLA` / `CTRLB`（AMODE/GCMD） | 同 | 同 |
| 0x0C | `BAUD`（32 位：BAUD+BAUDLOW+HS） | — | `BAUD`（8 位） | `BAUD`（16 位） |
| 0x14 / 0x16 / 0x18 | `INTENCLR` / `INTENSET` / `INTFLAG` | 同左，位名不同 | 同左 | 同左 |
| 0x1A / 0x1C | `STATUS` / `SYNCBUSY` | 同 | 同 | 同 |
| 0x24 | `ADDR`（找谁加 R/W） | `ADDR`（我是谁加掩码） | `ADDR`（从机地址加掩码） | — |
| 0x28 / 0x30 | `DATA` / `DBGCTRL` | 同 | 同 | 同 |

I2C 主机的 `INTFLAG` 是 MB/SB/ERROR，从机换成 PREC/AMATCH/DRDY/ERROR，SPI 换成 DRE/TXC/RXC——**偏移一样，位定义随 `MODE` 变**，这就是"同一块硬件、四种身份"的物证。所以 DFP 头文件把一台 SERCOM 写成一个大联合体（union），六个成员共用同一段地址：

```c
typedef union {
    sercom_i2cm_registers_t I2CM;  sercom_i2cs_registers_t I2CS;  sercom_spim_registers_t SPIM;
    sercom_spis_registers_t SPIS;  sercom_usart_int_registers_t USART_INT;
    sercom_usart_ext_registers_t USART_EXT;
} sercom_registers_t;
```

`SERCOM3_REGS->I2CM.SERCOM_CTRLA` 和 `SERCOM3_REGS->SPIM.SERCOM_CTRLA` 是**同一个物理寄存器**，只是走联合体的不同入口。用哪个入口，取决于你把 `MODE` 配成了什么。

> 代码口径：**本书统一使用 SAMD21_DFP 3.7.262 的写法**——实例宏是 `SERCOM3_REGS`，成员名是"外设_寄存器名"（`PORT_REGS->GROUP[0].PORT_DIRSET`、`SERCOM3_REGS->I2CM.SERCOM_CTRLA`），单个位的宏只有 `_Msk`（掩码）和 `_Val`（枚举值）两种形式。老版本 DFP / ASF 里的 `SERCOM3->I2CM.CTRLA.reg` 这种写法在本 DFP 中不存在，以你工程里的头文件为准。

### 7.2.3 六个 SERCOM 的地址与中断号

六个实例的基址是 `0x4200_0800 + 0x400 × n`（SERCOM0 到 SERCOM5，所以 SERCOM3 是 `0x4200_1400`），中断号 9 到 14，CORE 通用时钟 ID 是 `GCLK_CLKCTRL_ID_SERCOMx_CORE`。六个实例都在**桥 C**，所以第 2 章那条结论直接适用：`PM_APBCMASK` 复位后是关的，用之前必须先打开对应的 `PM_APBCMASK_SERCOMx_Msk`。

## 7.3 为什么 SERCOM 要两路通用时钟

第 2 章说过 SAMD21 的时钟分三层：振荡器 → GCLK 通用时钟 → 外设。SERCOM 特别的地方是它要**两路**通用时钟（手册 28.5.3）：

| 时钟 | 名字 | 谁需要它 | 为什么不能合并 |
|---|---|---|---|
| CORE | `GCLK_SERCOMx_CORE` | I2C 主机必配，它给波特率发生器提供 f_GCLK | 它决定 SCL 的频率，必须跟着总线速率走，可以很快（48 MHz） |
| SLOW | `GCLK_SERCOMx_SLOW` | 只在需要 SMBus 那套毫秒级超时时才配 | 手册 28.6.3.1：这些超时"由 GCLK_SERCOM_SLOW 驱动，必须配置成使用 32 kHz 振荡器" |

一个管**微秒级**的位时序，一个管**毫秒级**的超时判断，量级差三个数量级，所以硬件分成两路。本章例程只配 CORE，`LOWTOUTEN` / `SEXTTOEN` / `MEXTTOEN` 全部保持 0（复位默认），也就是不用那套硬件超时；等到写 SMBus 从机、或者想用硬件超时自动解开"从机拉住 SCL"的僵局，再把 SLOW 配上。

## 7.4 I2C 的硬件基础：开漏、上拉、线与

### 7.4.1 两根线，空闲都是高

I2C 只用两根线：**SDA**（Serial Data，数据）和 **SCL**（Serial Clock，时钟），空闲状态都是**高**。对 SERCOM 来说，这两根线固定挂在内部焊盘上（手册 28.4）：`PAD[0]` = SDA，`PAD[1]` = SCL，`PAD[2]`/`PAD[3]` = SDA_OUT/SCL_OUT（只在"四线模式"用，`CTRLA.PINOUT=1` 时旁路内部三态驱动器、要外接 I2C 驱动器）。本章用两线模式，`PINOUT` 保持 0。

### 7.4.2 开漏：只能拉低，不能主动拉高

第 4 章讲过，普通 GPIO 输出是**推挽**：上管接 VDD、下管接 GND，想输出高就开上管，想输出低就开下管。I2C 的输出级**只有下管**，这叫**开漏（open-drain）**：它只有两个状态——**拉低**，或者**放开**（高阻），永远不会主动输出高电平。

```text
   推挽（普通 GPIO）                开漏（I2C 的 SDA/SCL）
   VDD ─[上管]─┬─ 引脚            VDD ─[R 上拉]─┬─ 引脚
               │                                │
   GND ─[下管]─┘                  GND ──[下管]──┘
   高、低都能主动输出              只能拉低；高电平靠外接上拉电阻
```

于是"高电平"必须由**外部上拉电阻**提供。这是 I2C 硬件部分的第一个硬结论：**没有上拉电阻，I2C 一根线都不会动**，因为没有任何设备能把线拉高。

> SAMD21 的 PORT 里**没有开漏输出位**。手册修订记录专门写了一条 "I/O Pin Configuration: Removed reference to open-drain"。所以软件层面模拟开漏只有两态：`PORT_DIRSET` 加 `PORT_OUTCLR`（拉低）、`PORT_DIRCLR`（放开）。7.12.2 的总线恢复就靠这个。

### 7.4.3 线与：谁都能拉低，谁都不能独占

因为所有设备都只接了下管，一根线上的电平等价于"所有设备输出相与"：

| 设备 A | 设备 B | 线被谁决定 | 线上电平 |
|---|---|---|---|
| 放开 | 放开 | 上拉电阻 | 高 |
| 拉低 | 放开 | A 的下管 | 低 |
| 放开 | 拉低 | B 的下管 | 低 |
| 拉低 | 拉低 | 两边都在拉 | 低 |

这叫**线与（wired-AND）**。I2C 协议的三个关键机制都建立在它上面：**任何设备都能把时钟拉住不放（时钟拉伸）**、**从机可以拉低 SDA 表示 ACK**、**两个主机同时发数据时能检测出仲裁失败**。手册讲 SCL 时序时也是直接用 "wired-AND logic of the bus" 这个说法。

### 7.4.4 为什么不能推挽

两个理由。电气上：两个设备都用推挽时，一个输出高、一个输出低，就是 VDD 经两个开关直通 GND——第 4 章 4.1 节说的烧管子场景，在 I2C 上会变成每秒发生几十万次的常态。协议上：ACK 这一位就是**从机把 SDA 拉低**、**主机把 SDA 放开**，如果主机推挽输出高，这一位会被短路撕掉。所以 I2C 不是"也可以用开漏"，而是**必须开漏**。

### 7.4.5 上拉电阻取多大

上拉电阻被两头夹着：**太小**，下管灌电流超过 3 mA（手册 Table 37-16：VOL=0.4 V 时 IOL 上限 3 mA），低电平不够低；**太大**，上升沿太慢，还没到高电平就要采样了。下限是 R > (3.3 − 0.4) / 3 mA ≈ 970 Ω，上限由上升时间决定，用 I2C 标准的经验公式 t_r ≈ 0.8473 × R × C_b（t_r 定义在 0.3VDD 到 0.7VDD 之间）：

| 总线速率 | 允许 t_r（I2C 标准） | C_b = 100 pF | C_b = 200 pF |
|---|---|---|---|
| 100 kHz（标准模式） | ≈ 1000 ns | R ≤ 11.8 kΩ | R ≤ 5.9 kΩ |
| 400 kHz（快速模式） | ≈ 300 ns | R ≤ 3.5 kΩ | 1.8 kΩ（逼近下限，勉强） |

实务取值：**3.3 V、短线、100 kHz → 4.7 kΩ；400 kHz → 2.2 kΩ**。所以示波器上的 SCL 明显不对称：上升时间"由总线阻抗决定"，下降时间"由开漏电流上限和总线阻抗决定，通常可以当作 0"——**上升沿是弯的，下降沿是直的**。芯片内部的上下拉是 20–60 kΩ（手册 Table 37-15，Normal I/O Pins），比需要的值大十倍以上，**不能当 I2C 上拉用**。

## 7.5 一次 I2C 传输的完整流程

### 7.5.1 START 和 STOP 是两个"违规"的电平跳变

正常情况下 I2C 规定 **SCL 高的时候 SDA 必须稳定**（数据位只能在 SCL 低的时候改变）。START 和 STOP 故意违反这条规则，从而和数据位区分开：START 是 SCL 高时 SDA 由高变低，STOP 是 SCL 高时 SDA 由低变高；不发 STOP、直接再来一次 START 就是**重复 START**（Sr）。

```text
SCL  ‾‾‾‾‾‾‾‾‾╲______   ...   ______╱‾‾‾‾‾‾‾‾‾
SDA  ‾‾‾‾‾╲____________  ...  __________╱‾‾‾‾‾
       START：SCL 高时 SDA 下降      STOP：SCL 高时 SDA 上升
```

**重复 START** 的意义：主机要先写"寄存器指针"再读数据，中间不能放走总线（放了 STOP，别的事务可能插进来），所以用 Sr 代替 STOP。

### 7.5.2 一个字节 = 8 位数据 + 1 位应答

I2C 按字节走，**每 9 个时钟算一个包**：8 位数据（高位在前）加 1 位应答。手册 28.6.1 的原话是"每个 9 位数据包由 8 个数据位和 1 个应答位组成"。

| 顺序 | 内容 | 谁在驱动 SDA |
|---|---|---|
| START | SCL 高时 SDA 下降 | 主机 |
| 第 1 个字节 bit7…bit1 | 7 位从机地址 | 主机 |
| 第 1 个字节 bit0 | R/W：`0` = 写（主机到从机），`1` = 读（从机到主机） | 主机 |
| 第 9 个时钟 | 应答位 | 被寻址的从机 |
| 之后每 9 个时钟 | 8 位数据加 1 位应答 | 看方向 |
| STOP | SCL 高时 SDA 上升 | 主机 |

有个细节值得单独记住：**地址也是"数据"**。START 之后第一件事就是把地址当普通字节发出去，应答也是普通应答。所以从机地址写错一位，现象和"从机不存在"完全一样。

### 7.5.3 ACK 和 NACK 是谁发的

应答位永远是**接收方**发的，发法就是"把 SDA 拉低"：

| 方向 | 8 位数据之后 | 应答位由谁发 | ACK（拉低）表示 | NACK（放开）表示 |
|---|---|---|---|---|
| 主机写 | 主机发数据 | 从机 | 收到了，继续发 | 别再发了 / 我不在 |
| 主机读 | 从机发数据 | 主机 | 还要下一个字节 | 这是最后一个字节 |

主机的 NACK 有两种完全不同的含义，别混：**地址阶段收到 NACK** = 那个地址上没有设备；**数据阶段主机自己发 NACK** = 我读够了，你停。

### 7.5.4 主机发送和主机接收的完整流程

```text
写：START → 地址+W → ACK → 数据1 → ACK → 数据2 → ACK → … → STOP
读：START → 地址+R → ACK → 数据1 → ACK(主机发) → 数据2 → NACK(主机发) → STOP
                            ↑ 最后一定要 NACK，否则从机会一直发下去
```

### 7.5.5 每一步对应 STATUS / INTFLAG 的哪个位

I2C 主机的硬件被设计成**每个字节结束就刹车**：它把 SCL 拉住不放（时钟拉伸），置一个中断标志，然后等软件说下一步干什么。

| 软件动作 | 硬件在总线上做了什么 | 等哪个标志 | 结果看哪里 |
|---|---|---|---|
| 写 `ADDR`（地址+W） | 总线 IDLE 时发 START 加地址字节 | `INTFLAG.MB` | `STATUS.RXNACK=0` 表示从机 ACK；`=1` 表示没人应答 |
| 写 `DATA` | 把 1 个字节推出去 | `INTFLAG.MB` | `STATUS.RXNACK` |
| 写 `ADDR`（地址+R） | 总线是 OWNER 时自动插入重复 START 加地址 | `INTFLAG.SB`（收到字节）或 `MB`（地址被 NACK） | `STATUS.RXNACK` |
| 读 `DATA` | 取走刚收到的字节 | （SB 已经置着） | `STATUS.CLKHOLD=1` 表示时钟被拉住，数据有效 |
| 写 `CTRLB`（`ACKACT` 加 `CMD`） | 回 ACK/NACK，再读下一字节，或发 STOP | `INTFLAG.SB`（下一字节） | — |

三个标志的准确含义（手册 28.10.6）：`MB` = Host on Bus，**主机写模式下发完一个字节**（也包括仲裁丢失、总线错误）；`SB` = Client on Bus，**主机读模式下收到一个字节**；`ERROR` = 出现 `STATUS` 里任何一种错误。命名是个陷阱：`MB` / `SB` 直译是"主机/从机在总线上"，真实语义却是"写完成 / 读收到"，把它理解成"这个字节的交换结束了，轮到你决定"最省事。**写 `ADDR`、写 `DATA`、写一个有效的 `CMD`，都会自动把标志清掉**，因为它们正是"你决定了下一步"的动作；手册专门提醒，标志置位后如果不做这些动作，总线会一直卡在那里。`INTFLAG` 和 `STATUS` 的分工是：前者说"发生了一个字节事件"，后者说"这个事件的结果是什么"。

## 7.6 哪些引脚能做 I2C

### 7.6.1 手册 Table 7-5

不是所有引脚都能做 I2C。手册 7.2.3 节给了一张专门的表：

| 封装   | 支持 I2C 模式的引脚                                                                       |
| ---- | ---------------------------------------------------------------------------------- |
| 32 脚 | PA08, PA09, PA16, PA17, PA22, PA23                                                 |
| 48 脚 | PA08, PA09, PA12, PA13, PA16, PA17, PA22, PA23                                     |
| 64 脚 | PA08, PA09, PA12, PA13, PA16, PA17, PA22, PA23, PB12, PB13, PB16, PB17, PB30, PB31 |

> **ATSAMD21G18A 是 48 脚**（手册 Ordering Information：E = 32 脚、G = 48 脚、J = 64 脚），所以本章按 48 脚那一行看，是 8 个脚。**PA22/PA23 三行里都有**，所以例程选它，换封装不用改代码。

### 7.6.2 为什么只有这一部分引脚

回到 7.2.1 那张骨架图：SERCOM 只有 4 个内部焊盘，而 I2C 固定用 **PAD[0]=SDA、PAD[1]=SCL**。每个物理引脚能连到哪个 SERCOM 的哪个 PAD，是芯片布线时定死的，写在手册 Table 7-1 里。把 Table 7-1 里所有以 PAD[0]/PAD[1] 出现的引脚挑出来，正好就是 Table 7-5 那 14 个：

| 引脚 | SERCOM 功能 | 角色 | 引脚 | SERCOM 功能 | 角色 |
|---|---|---|---|---|---|
| PA08 | SERCOM0/PAD[0]、SERCOM2/PAD[0] | SDA | PB12 | SERCOM4/PAD[0] | SDA |
| PA09 | SERCOM0/PAD[1]、SERCOM2/PAD[1] | SCL | PB13 | SERCOM4/PAD[1] | SCL |
| PA12 | SERCOM2/PAD[0]、SERCOM4/PAD[0] | SDA | PB16 | SERCOM5/PAD[0] | SDA |
| PA13 | SERCOM2/PAD[1]、SERCOM4/PAD[1] | SCL | PB17 | SERCOM5/PAD[1] | SCL |
| PA16 | SERCOM1/PAD[0]、SERCOM3/PAD[0] | SDA | PB30 | SERCOM5/PAD[0] | SDA |
| PA17 | SERCOM1/PAD[1]、SERCOM3/PAD[1] | SCL | PB31 | SERCOM5/PAD[1] | SCL |
| PA22 | SERCOM3/PAD[0]、SERCOM5/PAD[0] | SDA | （其余引脚只连到 PAD[2]/PAD[3]） | — | 不能做 I2C |
| PA23 | SERCOM3/PAD[1]、SERCOM5/PAD[1] | SCL | — | — | — |

反过来说：**只连到 PAD[2]/PAD[3] 的引脚，做 USART/SPI 可以，做 I2C 永远不行**。比如 PA18/PA19 是 SERCOM1 的 PAD[2]/PAD[3]，可以当串口的 TX/RX；PA24/PA25 是 SERCOM3 的 PAD[2]/PAD[3]，同样做不了 I2C。这就是引脚限制的根源——不是软件没配好，是硬件没接过去。两条实用规则：**同一对 PAD[0]/PAD[1] 一般来自同一个 SERCOM**（PA22/PA23 都是 SERCOM3，或都是 SERCOM5）；**同一对引脚上选哪个 SERCOM，由 PMUX 的功能号决定**（PA22 的功能 C 是 SERCOM3，功能 D 是 SERCOM5）。

### 7.6.3 把引脚交给 SERCOM

引脚复位后归 GPIO 管（第 4 章）。要让它归 SERCOM，动两个 PORT 寄存器即可：

```c
/* PMUX[11] 管 PA22（偶数脚 → PMUXE）和 PA23（奇数脚 → PMUXO）；功能 C = SERCOM3 */
PORT_REGS->GROUP[0].PORT_PMUX[SDA_PIN / 2] = (PORT_PMUX_PMUXE_C_Val |
                                              (PORT_PMUX_PMUXO_C_Val << PORT_PMUX_PMUXO_Pos));
/* 打开复用开关，引脚从 GPIO 移交给 SERCOM */
PORT_REGS->GROUP[0].PORT_PINCFG[SDA_PIN] = PORT_PINCFG_PMUXEN_Msk;
PORT_REGS->GROUP[0].PORT_PINCFG[SCL_PIN] = PORT_PINCFG_PMUXEN_Msk;
```

交接之后**方向和电平都不用你管**了。手册 28.5.1 写得很清楚：I2C 模式下由 SERCOM 控制 I/O 引脚的方向和值。也就是说，开漏那两态是 SERCOM 自己切换的，你只需要保证外部有上拉电阻。

## 7.7 使用逻辑：初始化的六步，以及为什么是这个顺序

手册 28.6.2.1 给了初始化清单，但没说"为什么必须按这个顺序"。这里补上：

| 顺序 | 做什么 | 为什么必须在这个位置 |
|---|---|---|
| 1 | 打开 APB 时钟 `PM_APBCMASK_SERCOMx_Msk` | SERCOM 在桥 C，复位后总线时钟是关的。不开就先写寄存器，值会被**静默丢弃**（第 2 章那个坑） |
| 2 | 配 GCLK 的 `SERCOMx_CORE` | 波特率发生器靠它出 SCL。没有它，寄存器读写正常，总线上一个跳变都没有 |
| 3 | 把引脚交给 SERCOM（`PMUX` 加 `PINCFG.PMUXEN`） | 引脚默认归 GPIO。不交接，波形出不了芯片 |
| 4 | 在 `ENABLE=0` 时写 `CTRLA.MODE`、`CTRLA` 其它位、`BAUD`、`CTRLB` | 这些是 **Enable-Protected**：手册 28.6.2.1 明确列出 `CTRLA`（除 ENABLE/SWRST）、`CTRLB`（除 ACKACT/CMD）、`BAUD` 只能在关闭时写，使能后写会被丢弃 |
| 5 | `CTRLA.ENABLE=1`，等 `SYNCBUSY` | 使能要跨时钟域，不等就可能紧接着写坏下一笔 |
| 6 | 把 `STATUS.BUSSTATE` 强制成 IDLE | 使能后总线状态机是 **UNKNOWN**。手册 28.6.2.3 说得很直接：从 UNKNOWN 出来只有三条路——软件强制写 `BUSSTATE=0b01`、检测到 STOP、或者配了 inactive 超时。不强制，第一个 START 发不出去 |

一句话概括：**时钟 → 引脚 → 配置 → 使能 → 校准总线状态**。前两步让它能工作，第三步让它能接线，第四步告诉它做什么，第五、六步让它上路。顺序题外话：`ADDR` 在**从机身份下是 Enable-Protected**（那是"我的地址"，使能后不能改），在**主机身份下不是**——因为主机的 `ADDR` 每次写都是一条总线命令，必须能在运行时写。同一个偏移，两种保护属性。

## 7.8 寄存器逐个讲：每个寄存器回答什么问题

### 7.8.1 CTRLA：我是谁、开不开、要不要复位

`CTRLA` 在偏移 `0x00`，复位值 `0x00000000`。属性：PAC 写保护、Enable-Protected、写同步。

| 位 | 名字 | 回答什么问题 | 本章取值 |
|---|---|---|---|
| 4:2 | `MODE[2:0]` | 这个 SERCOM 是哪一种身份？ | `0x5` = I2C 主机（`SERCOM_I2CM_CTRLA_MODE_I2C_MASTER`） |
| 1 | `ENABLE` | 现在开始工作吗？ | 配置完最后置 1 |
| 0 | `SWRST` | 出问题了，把这台外设整个恢复到复位状态 | 平时 0；卡死时写 1 逃生 |
| 7 | `RUNSTDBY` | 睡眠时还要不要跑？ | 0（第 6 章再管） |
| 25:24 | `SPEED[1:0]` | 总线速率档位 | `0x0` = Sm/Fm（100 k / 400 k）；`0x1` = Fm+（1 MHz）；`0x2` = Hs（3.4 MHz） |
| 27 | `SCLSM` | 什么时候拉伸 SCL | `0` = ACK **之前**拉伸（软件先看数据再决定 ACK/NACK，默认，推荐） |
| 30 | `LOWTOUTEN` | SCL 被拉住 25–35 ms 要不要自动放弃？ | 0（要用 SLOW 时钟） |
| 21:20 | `SDAHOLD` | SDA 相对 SCL 下降沿保持多久 | 0 = 关闭 |
| 29:28 | `INACTOUT` | 总线静多久算空闲 | 0 = 关闭（SMBus 用） |
| 16 | `PINOUT` | 两线还是四线 | 0 = 两线 |

`SWRST` 值得单独说一句：它**优先于同一次写入里的其它位**，写 1 之后整个 SERCOM 回到复位状态并被关闭，`SYNCBUSY.SWRST` 会置位直到完成。这是"配置写乱了、总线卡死了"时的**逃生门**——不用复位整颗芯片，复位一台外设就够了。`SCLSM` 决定你写代码的节奏：手册 Figure 28-5 描述的是 `SCLSM=0`——**硬件在每个字节之后把 SCL 拉住，等你读完数据、写好 ACK/NACK 和下一步命令，才放开时钟继续**。这就是轮询代码里"等 SB → 读 DATA → 写 CMD"能一步不乱地走的原因。置 1 之后中断只在 ACK 之后发生（高速模式强制这样），软件必须提前把 `ACKACT` 准备好。本章用 `SCLSM=0`。

### 7.8.2 CTRLB：下一步做什么动作

`CTRLB` 在偏移 `0x04`，复位值 `0x00000000`。本章只用到三个位：`CMD[1:0]`（17:16，这个字节交换结束了**下一步做什么**，strobe 位、读出永远是 0）、`ACKACT`（18，收到字节之后回 ACK 还是 NACK）、`SMEN`（8，要不要"读 DATA 时自动发 ACK/NACK"）。

`CMD` 的四个值（手册 Table 28-4）：`0x0` 无动作；`0x1` 先执行应答动作再发**重复 START**；`0x2`（读方向）先执行应答动作再**读下一个字节**；`0x3` 先执行应答动作再发 **STOP**。`ACKACT`：`0` = 发 ACK，`1` = 发 NACK，它**什么时候生效**取决于 `SMEN`（手册 28.10.2 原话）：

| `SMEN` | 应答动作在什么时候执行 | 代码节奏 |
|---|---|---|
| 0（本章） | 写 `CTRLB.CMD` 的时候 | 每次都要显式写 `CMD`，每一步总线动作都看得见 |
| 1（智能模式） | **读 `DATA.DATA`** 的时候 | 少写一次 `CMD`，但"读数据"带副作用，调试时容易懵 |

`CMD` 和 `ACKACT` 可以**在同一次写里**给出，硬件会先更新应答动作再触发命令——这正是手册推荐的写法。注意 `CTRLB` 里还有 `SMEN`/`QCEN`：如果你开了 `SMEN`，之后每次写 `CTRLB` 都要把它带回去，否则下一次写就把它清掉了。本章 `SMEN=0`。

### 7.8.3 BAUD：SCL 的高、低各占多久

`BAUD` 在偏移 `0x0C`，是**32 位**的，里面塞了两套（普通模式和高速模式）各两个字节：bit7:0 是 `BAUD`，`BAUDLOW` 非 0 时它决定 **SCL 高电平**时长，`BAUDLOW` 为 0 时高低都用它；bit15:8 是 `BAUDLOW`，决定 **SCL 低电平**时长；bit23:16 和 bit31:24 是 `HSBAUD` / `HSBAUDLOW`，高速模式（`SPEED=0x2`）专用，本章不用。

为什么要高低分开？因为 I2C 的上升沿由 RC 充电决定（7.4.5），下降沿几乎瞬时，波形天然不对称；而且 Fm+ 明确要求高比低约为 1:2。手册给的公式是（`f_GCLK` 就是 `GCLK_SERCOMx_CORE`）：

```text
BAUDLOW = 0（复位默认）:  f_SCL = f_GCLK / (10 + 2 × BAUD + f_GCLK × t_RISE)
BAUDLOW ≠ 0:             f_SCL = f_GCLK / (10 + BAUD + BAUDLOW + f_GCLK × t_RISE)
T_LOW  = (BAUDLOW + 5) / f_GCLK        T_HIGH = (BAUD + 5) / f_GCLK
```

`BAUDLOW = 0` 是个**特例**，不是"低电平等于 5/f_GCLK"：手册明确说此时 `BAUD` 同时决定高低两段。想把高低设成不同值，才需要写 `BAUDLOW`。按 `f_GCLK = 48 MHz`、暂时忽略 `t_RISE` 来算：

| 目标 SCL | `SPEED` | `BAUD` | `BAUDLOW` | 验算 |
|---|---|---|---|---|
| 100 kHz | `0` | 235 | 0 | 48 M / (10 + 470) = 100 kHz |
| 400 kHz | `0` | 55 | 0 | 48 M / (10 + 110) = 400 kHz |
| 1 MHz（Fm+，高比低 = 1:2） | `1` | 11 | 27 | 高 16/48 M ≈ 333 ns，低 32/48 M ≈ 667 ns |

如果时钟还没按第 5 章配到 48 MHz，`f_GCLK` 是复位后的 1 MHz，同样公式算出来的 SCL 最高也只有 100 kHz——**I2C 用不了 400 kHz，不是寄存器写错，是通用时钟不够快**。

### 7.8.4 ADDR：写这个寄存器就是发一次 START

`ADDR` 在偏移 `0x24`。这是"一块硬件两种身份"最典型的地方：

| 身份 | `ADDR` 的含义 | 属性 |
|---|---|---|
| 主机（`MODE=0x5`） | **我要找谁**：bit10:0 是地址，bit0 兼作 R/W 位 | 写它就是触发一次总线动作（START / 重复 START 加地址），**不是** Enable-Protected |
| 从机（`MODE=0x4`） | **我是谁**：`ADDR.ADDR` 是 7/10 位本机地址，`ADDR.ADDRMASK` 是掩码 | Enable-Protected，使能后写不进去 |

主机身份下，写 `ADDR` 之后硬件根据当前总线状态决定做什么（手册 28.10.9）：总线是 **IDLE** → 发 START 再发地址；总线是 **OWNER**（我就是主人）→ 发**重复 START** 再发新地址；总线是 **BUSY**（别人在用）→ 等，直到总线变成 IDLE 再动；总线是 **UNKNOWN** → 置 `INTFLAG.MB` 加 `STATUS.BUSERR`，操作终止。地址怎么算？**7 位地址左移一位，最低位填 R/W**，这是新手最容易错的一步：

```text
从机 7 位地址 0x68 = 110 1000     写: 0xD0 = (0x68 << 1) | 0
                                  读: 0xD1 = (0x68 << 1) | 1
```

顺带一个福利：写 `ADDR` 会自动清掉 `STATUS.BUSERR`、`STATUS.ARBLOST`、`INTFLAG.MB`、`INTFLAG.SB`（手册 28.10.9）。所以"发一次新地址"天然就是"把上一次的残留清干净、重新开始"。

### 7.8.5 DATA：收发共用的一个口

`DATA` 在偏移 `0x28`，只有低 8 位有效。**写等于发送，读等于取走刚收到的字节**，靠上下文区分方向和角色。两个必须记住的限制：手册 28.10.10 说只有在 **`STATUS.CLKHOLD = 1`（SCL 被主机拉住）**时读写 `DATA` 才有效（唯一的例外是 STOP 之后读最后一个字节）；访问 `DATA` 会**自动触发**总线操作，写 `DATA` 就是把字节推出去，不需要再写什么命令。`CLKHOLD` 是硬件给软件的安全窗口：因为每个字节结束时 SCL 都被拉住，你从"标志置位"到"读完 `DATA`"这段时间里，总线是冻住的，不会跑掉。

### 7.8.6 INTFLAG、INTENCLR / INTENSET：事件标志和它的开关

`INTFLAG` 在偏移 `0x18`，8 位，只有三个位：`MB`（bit0）、`SB`（bit1）、`ERROR`（bit7，出现任何错误时置位，对应 `STATUS` 里的 `LENERR`/`SEXTTOUT`/`MEXTTOUT`/`LOWTOUT`/`ARBLOST`/`BUSERR`）。它叫 INTFLAG，但**不一定非要开中断**：轮询"标志位什么时候变成 1"，等价于"等这个事件发生"，本章例程就是这么做的，中断留给第 12 章。`INTENCLR`（0x14）和 `INTENSET`（0x16）是同一套位的中断使能开关：**写 1 关**、**写 1 开**，写 0 无效果。为什么不做成一个"中断使能寄存器"让你读-改-写？理由和第 4 章的 `OUTSET`/`OUTCLR` 完全一样：只改我想改的那一位，一条写指令搞定，天然原子（第 2 章 2.2.2 汇总过这套模式）。本章是纯轮询，这两个寄存器一次都不写。

### 7.8.7 STATUS：刚才那一步的结果

`STATUS` 在偏移 `0x1A`，16 位，**不受 PAC 写保护**（和 `DATA`、`ADDR`、`INTFLAG` 一样，这四个是手册列出的例外）。

| 位 | 名字 | 读它来回答什么问题 |
|---|---|---|
| 2 | `RXNACK` | 刚才那个地址/数据字节，对方**应答了吗**？（1 = NACK） |
| 5:4 | `BUSSTATE[1:0]` | 总线现在归谁：`00` UNKNOWN、`01` IDLE、`10` OWNER（我是主人）、`11` BUSY（别人在用） |
| 7 | `CLKHOLD` | 主机是不是正把 SCL 拉住？（只读） |
| 1 / 0 | `ARBLOST` / `BUSERR` | 仲裁输了（多主机才有） / 总线上出现了违反协议的 START、STOP 或超时 |
| 6 / 8 / 9 | `LOWTOUT` / `MEXTTOUT` / `SEXTTOUT` | SCL 低电平超时、主机累积拉伸超时、从机累积拉伸超时 |
| 10 | `LENERR` | DMA 自动长度传输里从机提前 NACK |

两个用法提醒：**`BUSSTATE` 可以写**，手册 28.6.2.3 说总线状态是 UNKNOWN 时写 `0b01` 可以强制它变成 IDLE，这就是初始化第 6 步的依据；**`BUSERR`/`ARBLOST`/`MEXTTOUT`/`SEXTTOUT`/`LOWTOUT`/`LENERR` 这些错误位写 `ADDR` 时会自动清掉**，不用手动挨个清。

### 7.8.8 SYNCBUSY：跨时钟域的等待

`SYNCBUSY` 在偏移 `0x1C`，只读，三位：`SWRST`（写完 `CTRLA.SWRST` 后置位）、`ENABLE`（写完 `CTRLA.ENABLE` 后置位）、`SYSOP`（写完 `CTRLB.CMD`、`STATUS.BUSSTATE`、`ADDR`、`DATA` 之后置位，同步完成前一直是 1）。原因还是第 2 章那条：通用时钟域和 APB 总线域不同步，写下去的值要"过一道"，所以代码里写成 `while (SERCOM3_REGS->I2CM.SERCOM_SYNCBUSY) { }` 三个位一起等最省事。

## 7.9 完整例程：读一个从机的两个寄存器

### 7.9.1 硬件需求

| 项目 | 取值 | 依据 |
|---|---|---|
| 总线引脚 | PA22 = SDA，PA23 = SCL | 手册 Table 7-5 加 Table 7-1（SERCOM3/PAD[0]、PAD[1]，功能 C） |
| 从机 7 位地址 / 寄存器 | `0x68` / 指针 `0x00` 起连读 2 字节 | 换成你自己器件的地址和寄存器 |
| 上拉电阻 | 4.7 kΩ × 2（100 kHz，短线） | 7.4.5；**必须外接**，内部 20–60 kΩ 不行 |
| 通用时钟 | `GCLK_SERCOM3_CORE` = `GCLK0` = 48 MHz | 第 5 章；没配到 48 MHz 就只能跑 100 kHz |

```text
要读 2 个字节 → 先写寄存器指针（写方向）→ 不发 STOP，改成重复 START 换读方向
             → 读第一字节 → ACK → 读第二字节 → NACK → STOP

寄存器顺序：PM_APBCMASK → GCLK_CLKCTRL → PMUX/PINCFG → CTRLA(MODE) → BAUD
          → ENABLE → STATUS.BUSSTATE=IDLE → ADDR(写) → DATA(指针) → ADDR(读) → DATA×2 + CMD
```

### 7.9.2 完整代码

```c
#include "sam.h"

#define I2C_ADDR     0x68u      /* 从机 7 位地址，换成你自己器件的 */
#define I2C_REG_PTR  0x00u      /* 要读的寄存器指针 */
#define SDA_PIN      22u        /* PA22 = SERCOM3/PAD[0] */
#define SCL_PIN      23u        /* PA23 = SERCOM3/PAD[1] */
#define I2C_TIMEOUT  200000u    /* 还没学定时器，超时先数圈数（第 9 章换掉） */

enum { I2C_OK = 0, I2C_ERR_TIMEOUT, I2C_ERR_NACK, I2C_ERR_BUS };

/* 通用时钟域和 APB 域不同步：SYNCBUSY 归零才代表这一笔写生效了 */
static void i2c_sync(void) { while (SERCOM3_REGS->I2CM.SERCOM_SYNCBUSY) { } }

/* 等"一个字节交换结束"。返回 INTFLAG 内容，返回 0 表示超时 */
static uint8_t i2c_wait(void)
{
    for (uint32_t i = 0; i < I2C_TIMEOUT; i++) {
        uint8_t f = SERCOM3_REGS->I2CM.SERCOM_INTFLAG;
        if (f & (SERCOM_I2CM_INTFLAG_MB_Msk | SERCOM_I2CM_INTFLAG_SB_Msk |
                 SERCOM_I2CM_INTFLAG_ERROR_Msk)) { return f; }
    }
    return 0u;      /* 典型原因：SCL 被从机拉住不放，或 CORE 通用时钟没配 */
}

/* 发 STOP（CMD=0x3），再等总线回到 IDLE，好让下一次事务立刻开始 */
static void i2c_stop_and_idle(void)
{
    SERCOM3_REGS->I2CM.SERCOM_CTRLB = SERCOM_I2CM_CTRLB_CMD(3);
    i2c_sync();
    for (uint32_t i = 0; i < I2C_TIMEOUT; i++) {        /* BUSSTATE=0b01 就是 IDLE */
        if ((SERCOM3_REGS->I2CM.SERCOM_STATUS & SERCOM_I2CM_STATUS_BUSSTATE_Msk)
            == SERCOM_I2CM_STATUS_BUSSTATE(1)) { return; }
    }
}

static void i2c_init(void)
{
    /* ① SERCOM3 在桥 C，APB 时钟复位后是关的，不开就白写 */
    PM_REGS->PM_APBCMASK |= PM_APBCMASK_SERCOM3_Msk;

    /* ② 波特率发生器靠 CORE 通用时钟出 SCL，这里用 GCLK0 = 48 MHz */
    GCLK_REGS->GCLK_CLKCTRL = GCLK_CLKCTRL_ID_SERCOM3_CORE |
                              GCLK_CLKCTRL_GEN_GCLK0 |
                              GCLK_CLKCTRL_CLKEN_Msk;
    while (GCLK_REGS->GCLK_STATUS & GCLK_STATUS_SYNCBUSY_Msk) { }

    /* ③ 把 PA22/PA23 从 GPIO 移交给 SERCOM3：PMUX 选功能 C */
    PORT_REGS->GROUP[0].PORT_PMUX[SDA_PIN / 2] = (PORT_PMUX_PMUXE_C_Val |
                                                  (PORT_PMUX_PMUXO_C_Val << PORT_PMUX_PMUXO_Pos));
    PORT_REGS->GROUP[0].PORT_PINCFG[SDA_PIN] = PORT_PINCFG_PMUXEN_Msk;
    PORT_REGS->GROUP[0].PORT_PINCFG[SCL_PIN] = PORT_PINCFG_PMUXEN_Msk;

    /* ④ 以下都是 Enable-Protected，必须趁 ENABLE=0 写完 */
    SERCOM3_REGS->I2CM.SERCOM_CTRLA = SERCOM_I2CM_CTRLA_MODE_I2C_MASTER |  /* MODE=0x5 */
                                      SERCOM_I2CM_CTRLA_SPEED(0);          /* Sm/Fm 档 */
    i2c_sync();
    SERCOM3_REGS->I2CM.SERCOM_BAUD = SERCOM_I2CM_BAUD_BAUD(235);  /* 48M/(10+2×235)=100kHz */
    SERCOM3_REGS->I2CM.SERCOM_INTFLAG = SERCOM_I2CM_INTFLAG_MB_Msk |
                                        SERCOM_I2CM_INTFLAG_SB_Msk |
                                        SERCOM_I2CM_INTFLAG_ERROR_Msk;  /* 清残留标志 */

    /* ⑤ 使能，等同步 */
    SERCOM3_REGS->I2CM.SERCOM_CTRLA |= SERCOM_I2CM_CTRLA_ENABLE_Msk;
    i2c_sync();

    /* ⑥ 使能后总线状态是 UNKNOWN，强制成 IDLE，否则第一个 START 发不出去 */
    SERCOM3_REGS->I2CM.SERCOM_STATUS = SERCOM_I2CM_STATUS_BUSSTATE(1);
    i2c_sync();
}

/* 从 addr 的 reg 寄存器开始，连读 n 个字节到 buf */
static int i2c_read_regs(uint8_t addr, uint8_t reg, uint8_t *buf, uint32_t n)
{
    uint8_t f;

    if (n == 0u) { return I2C_OK; }
    /* 清掉上一轮残留的标志，保证后面等到的都是本轮的事件 */
    SERCOM3_REGS->I2CM.SERCOM_INTFLAG = SERCOM_I2CM_INTFLAG_MB_Msk |
                                        SERCOM_I2CM_INTFLAG_SB_Msk |
                                        SERCOM_I2CM_INTFLAG_ERROR_Msk;

    /* ① 写 ADDR = 地址<<1 | 0。总线是 IDLE，这一步直接产生 START + 地址 */
    SERCOM3_REGS->I2CM.SERCOM_ADDR = ((uint32_t)addr << 1);
    i2c_sync();
    f = i2c_wait();
    if (f == 0u)                           { return I2C_ERR_TIMEOUT; }
    if (f & SERCOM_I2CM_INTFLAG_ERROR_Msk) { i2c_stop_and_idle(); return I2C_ERR_BUS; }
    if (SERCOM3_REGS->I2CM.SERCOM_STATUS & SERCOM_I2CM_STATUS_RXNACK_Msk) {
        i2c_stop_and_idle();    /* 地址被 NACK：这个地址上没有设备应答（手册建议发 STOP 收场） */
        return I2C_ERR_NACK;
    }

    /* ② 写 DATA = 寄存器指针，硬件把它推出去 */
    SERCOM3_REGS->I2CM.SERCOM_DATA = reg;
    i2c_sync();
    if (i2c_wait() == 0u) { return I2C_ERR_TIMEOUT; }
    if (SERCOM3_REGS->I2CM.SERCOM_STATUS & SERCOM_I2CM_STATUS_RXNACK_Msk) {
        i2c_stop_and_idle();
        return I2C_ERR_NACK;    /* 从机不收这个指针，多半是寄存器不存在 */
    }

    /* ③ 再写 ADDR，这次最低位是 1。总线已是 OWNER，硬件自动插入重复 START */
    SERCOM3_REGS->I2CM.SERCOM_ADDR = ((uint32_t)addr << 1) | 1u;
    i2c_sync();

    /* ④ 之后每收到一个字节，硬件把 SCL 拉住并置 SB，等软件决定下一步 */
    for (uint32_t i = 0; i < n; i++) {
        f = i2c_wait();
        if (f == 0u) { i2c_stop_and_idle(); return I2C_ERR_TIMEOUT; }
        if (f & SERCOM_I2CM_INTFLAG_MB_Msk) {   /* 读方向却等来 MB：地址被 NACK 了 */
            i2c_stop_and_idle();
            return I2C_ERR_NACK;
        }
        buf[i] = SERCOM3_REGS->I2CM.SERCOM_DATA;    /* CLKHOLD=1 期间读才是有效数据 */
        if (i + 1u < n) {
            /* 还要继续读：回 ACK（ACKACT=0），并开始接收下一字节（CMD=0x2） */
            SERCOM3_REGS->I2CM.SERCOM_CTRLB = SERCOM_I2CM_CTRLB_CMD(2);
        } else {
            /* 最后一个字节：回 NACK 再发 STOP（CMD=0x3），告诉从机别再发了 */
            SERCOM3_REGS->I2CM.SERCOM_CTRLB = SERCOM_I2CM_CTRLB_ACKACT_Msk |
                                              SERCOM_I2CM_CTRLB_CMD(3);
        }
        i2c_sync();
    }
    return I2C_OK;
}

int main(void)
{
    uint8_t two_bytes[2];
    i2c_init();
    /* 读 0x68 的 0x00、0x01 两个寄存器；出错看返回码：NACK = 地址或指针不对，
       TIMEOUT = 从机拉住时钟或 CORE 时钟没配 */
    i2c_read_regs(I2C_ADDR, I2C_REG_PTR, two_bytes, 2u);
    while (1) { }
}
```

### 7.9.3 这段代码在总线上到底发生了什么

```text
总线：  START → [0x68<<1|0]=0xD0 → ACK → [0x00] → ACK → Sr → 0xD1 → ACK → [数据1] → ACK → [数据2] → NACK → STOP
代码：  写ADDR ─等MB 查RXNACK→ 写DATA ─等MB 查RXNACK→ 写ADDR ──等SB──→ 读DATA 写CMD=2 ──等SB──→ 读DATA 写CTRLB(NACK+STOP)
```

对照 7.5.5 那张表读一遍，每一步都能找到出处。三个容易忽略的点：**第 ③ 步不是 STOP**，是重复 START——用 STOP 结束写方向就等于放开总线，别的事务可能插进来，从机的寄存器指针也就被覆盖了；**最后一次 `CTRLB` 写同时给出 `ACKACT=1` 和 `CMD=0x3`**，一次写里"回 NACK"加"发 STOP"，这就是手册说的"ACKACT 和 CMD 可以同时写"；**`i2c_sync()` 出现在每次写 `ADDR`/`DATA`/`CTRLB` 之后**，不写它多数时候也能跑，但在 48 MHz 下连着写，容易在 `SYSOP` 还没完成时覆盖上一笔，这是最难查的一类偶发问题。

## 7.10 初始化清单

以后遇到任何 SERCOM I2C 主机，按这个顺序想就不会漏：

| # | 动作 | 关键寄存器 |
|---|---|---|
| 1 | 开 APB 时钟 | `PM_REGS->PM_APBCMASK |= PM_APBCMASK_SERCOMx_Msk` |
| 2 | 配 CORE 通用时钟（要 SMBus 超时才加 SLOW） | `GCLK_REGS->GCLK_CLKCTRL` 加等 `GCLK_STATUS.SYNCBUSY` |
| 3 | 引脚交给 SERCOM | `PORT_PMUX[n]`（功能 C/D）加 `PORT_PINCFG[x].PMUXEN` |
| 4 | 选身份（必须在 `ENABLE=0` 时） | `CTRLA.MODE = 0x5` |
| 5 | 配速率（选 `SPEED` 加算 `BAUD`） | `CTRLA.SPEED`、`BAUD` |
| 6 | 清标志、使能、等同步 | `INTFLAG` 写 1 清、`CTRLA.ENABLE`、`SYNCBUSY` |
| 7 | 强制总线到 IDLE | `STATUS.BUSSTATE = 0b01` |

运行中的每一次事务，永远是同一个三段循环：**写 `ADDR`/`DATA`/`CMD` → 等 `INTFLAG` → 查 `STATUS.RXNACK`**。

## 7.11 一个 SERCOM I2C 主机的完整思维模型

```text
                        SERCOM3 当 I2C 主机
   时钟与供电 ──┬── 引脚 ────┬── 身份与速率 ──┬── 数据与状态
 PM_APBCMASK   PMUX 功能 C   CTRLA.MODE=0x5   ADDR 写 = 发 START
 GCLK CORE     PA22 = SDA    CTRLA.ENABLE=1   DATA 写=发 / 读=收
 (SLOW: SMBus) PA23 = SCL    BAUD = 高/低      INTFLAG MB/SB + STATUS
```

## 7.12 常见问题

### 7.12.1 全是 NACK 怎么排查

地址阶段的 NACK（写完 `ADDR` 就 `RXNACK=1`）只有三个可能，按这个顺序查：

| 检查点 | 具体怎么看 |
|---|---|
| 地址算错了 | 7 位地址要左移一位。`0x68` 发出去应该是 `0xD0`（写）或 `0xD1`（读）。很多器件手册给的是**已经左移过的 8 位地址**，再 `<<1` 就变成了两倍 |
| 从机根本没应答 | 器件没供电、没接 GND、复位脚悬空、地址脚（A0/A1/A2）接法和你想的不一样 |
| 电气不对 | 上拉电阻装了没有？SDA/SCL 有没有接反？总线电压和从机要求是否一致（1.8 V 器件挂 3.3 V 总线不会应答） |

数据阶段的 NACK 是另一回事：从机收下了地址但拒绝这个数据，比如写了一个不存在的寄存器指针，或者 EEPROM 正在写周期里。

### 7.12.2 总线卡死怎么恢复

**卡死的典型样子**：`STATUS.BUSSTATE = 0b11`（BUSY）或者始终不是 IDLE；某根线一直低；主机写什么都没反应。成因通常是上一次传输被打断（MCU 复位、调试器暂停、从机被热插拔），从机抱着 SDA 或 SCL 不放。先判断是谁拉住：**SCL 低** = 从机在做时钟拉伸或者状态机乱套；**SDA 低** = 从机输出到一半被打断。两种情况都可以用同一招：**把时钟手动敲够，让从机把剩下的位移完**。

```c
/* 总线恢复思路：把 SCL 当普通 GPIO 手动发 9 个脉冲，再补一个 STOP */
static void i2c_bus_recover(void)
{
    uint32_t m = (1UL << SCL_PIN);

    /* ① SWRST 把 SERCOM 复位并关闭，引脚交回 GPIO；这一步同时清掉 BUSSTATE */
    SERCOM3_REGS->I2CM.SERCOM_CTRLA = SERCOM_I2CM_CTRLA_SWRST_Msk;
    while (SERCOM3_REGS->I2CM.SERCOM_SYNCBUSY & SERCOM_I2CM_SYNCBUSY_SWRST_Msk) { }

    /* ② SCL 回到 GPIO 并配成"输入 + 上拉"：只放开，不驱动 */
    PORT_REGS->GROUP[0].PORT_PINCFG[SCL_PIN] = PORT_PINCFG_INEN_Msk |
                                               PORT_PINCFG_PULLEN_Msk;
    PORT_REGS->GROUP[0].PORT_DIRCLR = m;
    PORT_REGS->GROUP[0].PORT_OUTSET = m;      /* OUT=1 → 上拉 */

    /* ③ 敲 9 个脉冲：每个脉冲 = 放开（上拉拉高）+ 拉低 */
    for (int i = 0; i < 9; i++) {
        PORT_REGS->GROUP[0].PORT_DIRCLR = m;  /* 放开，上拉把 SCL 拉高 */
        delay_us(5);                          /* 自己实现微秒延时，见第 9 章 */
        PORT_REGS->GROUP[0].PORT_OUTCLR = m;  /* 先备好低电平（第 4 章 4.8 的顺序） */
        PORT_REGS->GROUP[0].PORT_DIRSET = m;  /* 再打开输出驱动 → 拉低 */
        delay_us(5);
    }
    /* ④ 补一个 STOP：SDA 先拉低，放开 SCL，再放开 SDA（同样只拉低/放开） */
    /* ⑤ 引脚交回 SERCOM3（重写 PMUX/PINCFG），再跑一遍 i2c_init() */
}
```

几个要点：**只能拉低或放开，绝不能驱动高**（推挽输出高会和从机的下管对打）；延时用 5 µs 量级（相当于 100 kHz）；恢复完必须重新 `i2c_init()`，因为 `SWRST` 已经把 `MODE`、`BAUD`、`ENABLE` 全清了。如果敲完 9 个脉冲总线还是低，那就是硬件问题（短路、上拉缺失、器件损坏），软件再怎么写也没用。

### 7.12.3 上拉电阻和速率怎么定

上拉取值见 7.4.5 的推导，结论是：3.3 V 短线 100 kHz 用 4.7 kΩ，400 kHz 用 2.2 kΩ，1.8 V 总线用 1.5 k–2.2 kΩ（下限随电压降低而变小）。排除法：**用芯片内部上拉（20–60 kΩ）一定不行**，**不接上拉一定不行**，**两个设备各接一个上拉不是"双重上拉"**而是并联、阻值减半（两个 4.7 kΩ 并联 = 2.35 kΩ，通常仍安全）。速率的配法是三步：确认 `f_GCLK`（`GCLK_SERCOMx_CORE` 的实际频率）→ 按 7.8.3 的公式算 `BAUD` → 选 `CTRLA.SPEED` 档位。

| 目标 | `SPEED` | `BAUD`（48 MHz 时） |
|---|---|---|
| 100 kHz | `0` | 235 |
| 400 kHz | `0` | 55 |
| 1 MHz | `1`（Fm+） | 11，`BAUDLOW` = 27 |

三个常见错误：**忘了 `SPEED`**（400 kHz 以下其实不影响，但 Fm+ 必须置 1）；**`BAUD` 算错单位**（公式里的 f_GCLK 是 Hz，不是 MHz）；**忽略 t_RISE**（`BAUD` 只有 8 位，400 kHz 时已经接近上限，上升沿慢的总线要把 `BAUD` 再调小一档）。

### 7.12.4 其它现象对照表

| 现象 | 最可能的原因 | 检查什么 |
|---|---|---|
| 写寄存器读回全 0 / 没反应 | 桥 C 的 APB 时钟没开 | `PM_APBCMASK` 里 `SERCOMx` 那一位 |
| 配置读回是对的，但总线上没有任何波形 | CORE 通用时钟没配 | `GCLK_CLKCTRL`、`GCLK_STATUS` |
| `MODE` 改了但没生效 | 在 `ENABLE=1` 时写的（Enable-Protected） | 先写 `CTRLA.ENABLE=0` |
| 第一个 START 发不出去 | 总线状态还是 UNKNOWN | 强制 `STATUS.BUSSTATE=0b01` |
| 等标志等到超时 | 从机在拉伸时钟，或时钟没跑 | `STATUS.CLKHOLD`、`STATUS.BUSSTATE` |
| 读出来的永远是 `0xFF` | SDA 没人驱动（上拉把它拉高） | 地址、寄存器指针、从机是否应答 |
| 偶发丢数据 | 没等 `SYNCBUSY` 就写下一次 | 每次写 `ADDR`/`DATA`/`CTRLB` 后加 `i2c_sync()` |

## 7.13 寄存器速查

I2C 主机（`MODE=0x5`）用到的寄存器，偏移以 SERCOM3 基址 `0x4200_1400` 为例：

| 偏移 | 名字 | 位 | 作用 |
|---|---|---|---|
| 0x00 | `CTRLA` | 4:2 `MODE`；1 `ENABLE`；0 `SWRST`；27 `SCLSM`；25:24 `SPEED`；30 `LOWTOUTEN` | 身份加使能加速率档位 |
| 0x04 | `CTRLB` | 17:16 `CMD`；18 `ACKACT`；8 `SMEN` | 下一步动作加应答动作 |
| 0x0C | `BAUD` | 7:0 `BAUD`；15:8 `BAUDLOW`；23:16 `HSBAUD`；31:24 `HSBAUDLOW` | SCL 高低时长 |
| 0x14 / 0x16 | `INTENCLR` / `INTENSET` | 0 `MB`、1 `SB`、7 `ERROR`（写 1 关 / 写 1 开） | 中断使能开关 |
| 0x18 | `INTFLAG` | 0 `MB`、1 `SB`、7 `ERROR`（写 1 清除） | 字节事件标志 |
| 0x1A | `STATUS` | 2 `RXNACK`；5:4 `BUSSTATE`；7 `CLKHOLD`；1 `ARBLOST`；0 `BUSERR`；6 `LOWTOUT`；8 `MEXTTOUT`；9 `SEXTTOUT`；10 `LENERR` | 上一步的结果 |
| 0x1C | `SYNCBUSY` | 0 `SWRST`、1 `ENABLE`、2 `SYSOP`（只读） | 跨时钟域等待 |
| 0x24 | `ADDR` | 10:0 `ADDR`（bit0 = R/W）；15 `TENBITEN`；14 `HS`；13 `LENEN`；23:16 `LEN` | 找谁加触发 START |
| 0x28 | `DATA` | 7:0 `DATA`（只低 8 位有效） | 写等于发，读等于收 |
| 0x30 | `DBGCTRL` | 0 `DBGSTOP` | 调试暂停时是否停波特率 |

`CMD` 速记：`0x1` = 重复 START，`0x2` = 读下一字节，`0x3` = STOP。`BUSSTATE`：`00` UNKNOWN、`01` IDLE、`10` OWNER、`11` BUSY。

## 7.14 最重要的几句话

1. SERCOM 是"一个引擎、多种身份"：`CTRLA.MODE` 决定它是 USART、SPI 还是 I2C，一旦使能资源就被独占。
2. 同一段地址上的寄存器，换个 `MODE` 就换个含义——主机的 `ADDR` 是"找谁"，从机的 `ADDR` 是"我是谁"。
3. I2C 必须开漏加外接上拉：设备只能拉低或放开，高电平由上拉电阻提供，所以没有上拉就没有 I2C。
4. 线与意味着任何设备都能拉低总线，这正是 ACK、时钟拉伸和仲裁检测的硬件基础。
5. 只有连到 SERCOM PAD[0]（SDA）和 PAD[1]（SCL）的引脚才能做 I2C；只连 PAD[2]/PAD[3] 的引脚永远不行。
6. 主机每完成一个字节就把 SCL 拉住并置标志：`MB` = 写完成，`SB` = 读收到，然后把决定权交回软件。
7. 写 `ADDR` 就是发 START（总线是主人时自动变成重复 START），bit0 是 R/W 位。
8. 配置顺序固定为**时钟 → 引脚 → 配置（ENABLE=0）→ 使能 → 强制 BUSSTATE 到 IDLE**，这个顺序由硬件属性逼出来。

## 7.15 自测

1. 为什么 SAMD21 要用"可配置串口"而不是像 STM32 那样给每个协议一套固定外设？代价是什么？
2. I2C 为什么必须用开漏输出？如果把 SDA 配成推挽输出，会发生什么？ACK 位还能正常工作吗？
3. 手册 Table 7-5 里有 14 个引脚支持 I2C，为什么偏偏是这 14 个？用 PAD 的概念解释。
4. 写完 `ADDR = (0x68 << 1) | 1` 之后，硬件具体会做什么？为什么这里不需要先手动发一个 START？
5. `INTFLAG.MB` 和 `INTFLAG.SB` 分别在什么情况下置位？为什么它们置位时 SCL 会被拉住？
6. `CTRLB.CMD = 0x2` 和 `CMD = 0x3` 有什么区别？在读两字节的例程里，哪一步该用哪个？
7. 主机初始化为什么不能先写 `CTRLA.MODE` 再开 GCLK？把顺序颠倒会出现什么现象？

## 附录：以后查手册怎么找 SERCOM I2C

```text
① Table 7-5（7.2.3） → 这个封装上哪些脚能做 I2C
② Table 7-1（7.1）   → 这个脚属于哪个 SERCOM 的哪个 PAD
③ 28.9 / 28.10       → 主机寄存器总览 + 逐个位定义
④ 28.6.2.4           → 主机行为（地址包四种结局、收发数据包）
⑤ 25.6.2.1 加第 15 章 → MODE 取值表、SERCOMx_CORE/SLOW 时钟 ID
⑥ 第 37/40/41 章     → IOL / VIL / VIH、上拉取值范围
```

几个定位技巧：手册里 I2C 叫 **I2C 主机 / 从机**（Host / Client），而 DFP 头文件的宏还叫 **MASTER / SLAVE**，两边说的是同一件事（`MODE=0x5` 既是 Host 也是 `SERCOM_I2CM_CTRLA_MODE_I2C_MASTER`）；寄存器总览先看 **Offset**（0x00 是 `CTRLA`、0x28 是 `DATA` 这条规律三种模式通用），再看位定义；每个寄存器描述开头的 **Property** 一行决定了你的写入姿势（Enable-Protected 就先关外设，Write-Synchronized 就等 `SYNCBUSY`）。

## 本章结束

学完这一章，`SERCOM3_REGS->I2CM.SERCOM_ADDR = (0x68 << 1) | 1;` 不该被读成"往某个寄存器写个数"，而应该读成：**在 I2C 总线上发一个 START，把 7 位地址 0x68 和"读"的方向位依次推出去，然后等从机在第九个时钟把 SDA 拉低回答我**。能把寄存器操作翻译回总线上的电平动作，I2C 才算真正学会。
