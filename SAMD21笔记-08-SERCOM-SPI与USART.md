
# 第 8 章 SERCOM（二）SPI 与 USART

> **芯片**：ATSAMD21G18A（Cortex-M0+，48 脚封装，GCLK0 = 48 MHz）
> **本章目标**：把 SERCOM 剩下的两种模式用完——SPI（主机 / 从机）和 USART（异步串口），并且彻底搞清"一个引脚怎么交给 SERCOM"。
> **学习顺序**：SERCOM 引擎 → SPI 硬件 → SPI 的使用逻辑 → SPI 寄存器 → USART 硬件 → USART 的使用逻辑 → USART 寄存器 → 引脚交给 SERCOM → 例程一（SPI Flash ID）→ 例程二（USART 回环）→ 初始化清单 → 思维模型 → 常见问题 → 速查 → 自测 → 附录。
> **前置**：第 2 章（时钟、使能、SYNCBUSY 的写法）、第 4 章（PORT）、第 5 章（把主频提到 48 MHz）、第 7 章（SERCOM 为什么三合一、I2C）。

## 8.0 先看 SERCOM 里有什么

I2C、SPI、USART 三种模式共用同一个**串行引擎**。它只有四块东西：

```text
发送：DATA(写) → 发送缓冲(1 级) → 移位寄存器 → 引脚
接收：引脚 → 移位寄存器 → 接收缓冲(2 级) → DATA(读)
                          ↑                    ↑
                          └── 波特率发生器 ← GCLK_SERCOMx_CORE
```

| 部件 | 作用 | 对软件的意义 |
|---|---|---|
| 发送缓冲（TxDATA）+ 移位寄存器 | 装"下一个要发的字节"，再一位一位挪出去 | 缓冲空时 `INTFLAG.DRE=1`，可以再写一个 |
| 接收缓冲（2 级） | 装已经收完、还没被读走的字节 | 有货时 `INTFLAG.RXC=1`，读 `DATA` 取走 |
| 波特率发生器 | 产生"一位有多长"的时钟 | SPI 主机和 USART 靠它，调节点是 `BAUD` 寄存器 |

"2 级接收缓冲"意味着硬件最多替你存两个字节，第 3 个挤进来就**溢出**（`STATUS.BUFOVF`）。所以软件必须在一个字节时间内把数据取走，或者接受丢数据——这是后面所有"读得不够快"问题的根源。

三种模式的差别只在"引脚上的时序"和"谁提供时钟"：

| 模式 | 时钟从哪来 | 引脚 | `CTRLA.MODE` |
|---|---|---|---|
| USART 异步 | 收发双方各自用 `BAUD` 生成 | TxD、RxD | 0x1 |
| SPI 从机 | 外部主机从 SCK 送进来 | MOSI、MISO、SCK、SS | 0x2 |
| SPI 主机 | SERCOM 自己用 `BAUD` 生成 SCK | 同上 | 0x3 |

（I2C 的 0x4 / 0x5 在上一章。）一个 SERCOM 实例**同一时刻只能是一种模式**：`CTRLA.MODE` 写下去、`ENABLE` 打开之后，整个引擎的资源就都归这个模式了。

> 名词对照：2026 版手册把 master/slave 改写成 **host / client**，含义不变；DFP 头文件里的宏仍然是 `SPI_MASTER` / `SPI_SLAVE`。本笔记按手册叫"主机 / 从机"。

### 8.0.1 代码里的寄存器写法（先对一次账）

**本书统一使用 SAMD21_DFP 3.7.262 的写法**：外设名 + `_REGS` 是基址宏，成员名是"外设_寄存器名"；SERCOM 要先选身份（`SPIM` / `SPIS` / `USART_INT` / `USART_EXT` / `I2CM` / `I2CS`）再访问 `SERCOM_xxx`；单个位域的宏只有 `_Msk`（掩码）形式，多位字段用已经移位好的可读宏。

网上大量教程用的是老式 ASF 写法（`.reg` / `.bit` 成员），对照关系是：

| 老式 ASF 写法 | DFP 3.7.262 的写法 |
|---|---|
| `PORT->Group[0].DIRSET.reg` | `PORT_REGS->GROUP[0].PORT_DIRSET` |
| `PORT->Group[0].PINCFG[16].reg` | `PORT_REGS->GROUP[0].PORT_PINCFG[16]` |
| `SERCOM1->CTRLA.reg` | `SERCOM1_REGS->SPIM.SERCOM_CTRLA`（SPI）/ `SERCOM1_REGS->USART_INT.SERCOM_CTRLA`（USART 内部时钟） |
| `GCLK->CLKCTRL.reg` | `GCLK_REGS->GCLK_CLKCTRL` |

宏名本身两套通用（`PORT_PMUX_PMUXE_C`、`SERCOM_SPIM_CTRLB_CHSIZE_8_BIT` 都是真名）。有一点要注意：**多位字段不要直接写 `_Val`**。`SERCOM_SPIM_CTRLA_MODE_SPI_MASTER_Val` 只是 3，要放到 bit4:2 上才等于主机模式；用带移位的 `SERCOM_SPIM_CTRLA_MODE_SPI_MASTER` 才不会错。

## 8.1 SPI 硬件：四根线，两种角色

SPI 是"主机说了算"的同步串行总线：主机产生时钟 SCK，双方在每个时钟周期里同时移出一位、移入一位。

```text
SPI 主机                                 SPI 从机
  MOSI ───────── 数据：主机 → 从机 ─────────→ MOSI
  MISO ←──────── 数据：从机 → 主机 ──────────  MISO
  SCK  ───────── 时钟：只有主机产生 ─────────→ SCK
  SS   ───────── 片选：低电平选中 ──────────→ SS
```

| 引脚 | 主机方向 | 从机方向 | 说明 |
|---|---|---|---|
| MOSI | 输出 | 输入 | 主机发给从机的数据 |
| MISO | 输入 | 输出 | 从机回给主机的数据；SS 为高时从机把它置成高阻（三态） |
| SCK | 输出 | 输入 | 时钟，一个周期一位 |
| SS | 输出 | 输入 | 低电平选中；`CTRLB.MSSEN=0` 时由软件用普通 GPIO 控制 |

由此得到一条贯穿全章的性质：**发和收是同一次动作的两面**。主机想读一个字节，就必须同时写一个字节（写什么从机不看）；反过来，主机写出去的时候 MISO 上一定会同时进来一个字节，想不丢就得把接收器打开并及时读走。

### 8.1.1 四种时钟模式：数据在哪个边沿动

时钟空闲时是高还是低，由 `CTRLA.CPOL` 决定；数据在第一个边沿采样还是第二个边沿采样，由 `CTRLA.CPHA` 决定。两者组合出四种模式：

| 模式 | CPOL | CPHA | 前沿（每个周期的第一个边沿） | 后沿 |
|---|---|---|---|---|
| 0 | 0 | 0 | 上升沿采样 | 下降沿变化 |
| 1 | 0 | 1 | 上升沿变化 | 下降沿采样 |
| 2 | 1 | 0 | 下降沿采样 | 上升沿变化 |
| 3 | 1 | 1 | 下降沿变化 | 上升沿采样 |

主机和从机必须用同一个模式。绝大多数 SPI Flash、传感器用模式 0 或模式 3（两者都是"第一个边沿采样"）。

### 8.1.2 SAMD21 的特殊之处：四根线要先落在 4 个 PAD 上

SERCOM 内部并不直接连引脚，而是有 4 个抽象接线柱，叫 PAD[0]~PAD[3]。哪根信号接哪个 PAD，由 `CTRLA` 的两个字段决定：

| 字段 | 决定什么 | 取值 |
|---|---|---|
| `CTRLA.DOPO`（输出） | 主机时的 MOSI(DO)、SCK、SS 的位置 | 0x0：DO=PAD0、SCK=PAD1、SS=PAD2；0x1：DO=PAD2、SCK=PAD3、SS=PAD1；0x2：DO=PAD3、SCK=PAD1、SS=PAD2；0x3：DO=PAD0、SCK=PAD3、SS=PAD1 |
| `CTRLA.DIPO`（数据输入 DI） | DI 落在哪个 PAD | 0x0~0x3 分别 = PAD[0]~PAD[3]；主机时 DI 就是 MISO，从机时 DI 是 MOSI |

`CTRLB.MSSEN=0` 时 SS 不由 SERCOM 驱动，`DOPO` 里那个 SS 位置就空着，你用一根普通 GPIO 代替。

### 8.1.3 SPI 从机的三个要点

从机不能主动开始传输，它的一切由 SS 和 SCK 决定，因此有三件事和主机不同：

| 要点 | 现象 | 开关 |
|---|---|---|
| SS 为高时 MISO 三态 | 没被选中的从机不会去驱动总线 | 硬件自动 |
| 第一个字节可能不是刚写进去的 | 上一个事务留在移位寄存器里的旧数据会被发出去 | `CTRLB.PLOADEN=1`：SS 为高时写 `DATA`，数据直接进移位寄存器（预装） |
| 可以用地址匹配代替一根片选 | `CTRLA.FORM=0x2` 时，事务的第一个字节和本机 `ADDR` 比较，不匹配就整段忽略 | `ADDR`、`ADDRMASK`、`CTRLB.AMODE` |

另外 `CTRLB.SSDE=1` 让 SS 的下降沿能把 CPU 从睡眠里叫醒（低功耗才需要），对应标志是 `INTFLAG.SSL`。除此之外从机很省心：引脚方向和 SS、SCK 的采样全是硬件做的，软件只负责"`DRE` 时喂数据、`RXC` 时取数据"。

## 8.2 使用逻辑：SPI 主机要做哪些准备

顺序不是随便定的，它由两条硬规则逼出来：`CTRLA`、`CTRLB`、`BAUD`、`ADDR` 都是**使能保护**的（只能在 `ENABLE=0` 时写，否则写进去被丢弃）；引脚必须先归 SERCOM 管，信号才出得来。

```text
① 时钟：PM 开 APB 时钟 → GCLK 把 GCLK_SERCOMx_CORE 接到 GCLK0
② 引脚：PORT 里 PINCFG.PMUXEN=1 + PMUX 选功能 C（外加 CS 用普通 GPIO）
③ 模式帧格式：CTRLA 的 MODE / DOPO / DIPO / CPOL / CPHA / DORD
④ 帧长与接收器：CTRLB 的 CHSIZE、RXEN
⑤ 波特率：BAUD（只有主机需要）
⑥ 使能：CTRLA.ENABLE=1，等 SYNCBUSY.ENABLE 清零
⑦ 收发：只在 INTFLAG.DRE=1 时写 DATA，只在 INTFLAG.RXC=1 时读 DATA
```

一次主机传输在时间轴上长这样：

```text
CS 拉低 ─┬─────────────────────────────────────────────┬─ CS 拉高
         └ 写 DATA → 8 个 SCK → 写 DATA → 8 个 SCK → … ┘
             ↑DRE=1 才能写      ↑RXC=1 就能读上一字节
```

注意最后一条：CS 的拉低和拉高**不属于 SERCOM 的配置**，它是软件在"事务"外面做的事。什么时候拉、什么时候放，决定了从机眼里"这是一次事务"还是"两次事务"——SPI Flash 的读命令如果被拆成两次事务，地址就丢了。

## 8.3 SPI 寄存器逐个讲

到这一节才看寄存器。每个寄存器都回答一个具体问题。

### CTRLA：我是谁、线在哪、什么时序（偏移 0x00）

| 位 | 名字 | 回答什么问题 |
|---|---|---|
| 4:2 | `MODE` | 主机还是从机？0x3=SPI 主机，0x2=SPI 从机 |
| 30 | `DORD` | 先发最高位还是最低位？0=MSB 先（SPI 常规） |
| 29 | `CPOL` | 空闲时 SCK 是高还是低 |
| 28 | `CPHA` | 第一个边沿采样还是第二个边沿采样 |
| 21:20 | `DIPO` | 数据输入 DI 在哪个 PAD |
| 17:16 | `DOPO` | DO、SCK（以及硬件 SS）在哪些 PAD |
| 27:24 | `FORM` | 帧格式：0x0 普通 SPI；0x2 从机地址匹配 |
| 8 | `IBON` | 接收溢出什么时候报：0 跟着数据流走，1 立即报 |
| 1 / 0 | `ENABLE` / `SWRST` | 总开关（写 1 后等 `SYNCBUSY.ENABLE`）/ 软复位整个 SERCOM |

### CTRLB：一次搬几位、接收器开不开（0x04）

| 位 | 名字 | 回答什么问题 |
|---|---|---|
| 2:0 | `CHSIZE` | 一次搬几位：0x0=8 位，0x1=9 位 |
| 17 | `RXEN` | 接收器开关。主机要读回数据就必须打开（收发共用同一次传输） |
| 13 | `MSSEN` | SS 由硬件控制吗？1=SERCOM 在 PAD 上自动拉低拉高，0=你自己用 GPIO |
| 6 | `PLOADEN` | 从机：预装移位寄存器，避免发出旧数据 |
| 15:14 | `AMODE` | 从机：地址匹配方式（掩码 / 两个地址 / 地址段） |
| 9 | `SSDE` | 从机：SS 下降沿唤醒使能 |

`CTRLB` 是**写同步**的：SERCOM 已使能时写它，要等 `SYNCBUSY.CTRLB=0`；在 `SYNCBUSY.CTRLB=1` 期间再写会产生 APB 错误（在 MPLAB X 里常常表现为莫名其妙的 HardFault 或复位）。

### BAUD：主机才用（0x0C，8 位）

SPI 走的是同步模式，手册 Table 25-2 给的同步公式是：

```text
f_SCK = f_ref / (2 × BAUD + 1)        BAUD = f_ref / (2 × f_SCK) − 1
```

48 MHz、想要约 4 MHz：`BAUD = 48/(2×4) − 1 = 5`，实际 `f_SCK = 48/11 ≈ 4.36 MHz`。BAUD 只能取整数，算出来不是整数时实际时钟必然偏离——Flash 通常能接受几十 MHz，所以这里留有余量就行。从机不需要 `BAUD`：时钟从 SCK 进来。

### DATA：读写同一个地址（0x28）

写 = 把字节放进发送缓冲（顺带触发一次传输），读 = 取走接收缓冲里的字节。9 位帧时用 `DATA[8]`。手册的要求很直接：**只在 `INTFLAG.DRE=1` 时写，只在 `INTFLAG.RXC=1` 时读。**

### INTFLAG / STATUS：三个"什么时候能碰 DATA"的标志（0x18 / 0x1A）

| INTFLAG 位 | 名字 | 什么时候置 1 | 怎么清 |
|---|---|---|---|
| 0 | `DRE` | 发送缓冲空，可以写下一个字节 | 写 `DATA` |
| 1 | `TXC` | 主机：最后一位发完且没有新数据；从机：SS 被拉高 | 写 1，或写新 `DATA` |
| 2 | `RXC` | 接收缓冲里有没读走的字节 | 读 `DATA` |
| 3 | `SSL` | 从机：SS 出现下降沿（需 `SSDE=1`） | 写 1 |
| 7 | `ERROR` | 有错（对应 `STATUS.BUFOVF`） | 写 1 |

`STATUS.BUFOVF`（bit2）是 SPI 唯一的错误位：接收缓冲满了，又来了一个字节。清法是写 1；`IBON=0` 时溢出信息跟着数据流走，读到那一笔数据时它才置起来，并伴随 `INTFLAG.ERROR`（手册 27.6.2.7）。

`DRE` 和 `TXC` 的区别值得单独记一句：`DRE` 说"缓冲区空了，还能再塞一个"，`TXC` 说"最后一个字节已经完整地出去了，而且没有新的了"。**要确认一段数据真的发完（准备拉高 CS、准备关发送器），等的是 `TXC`；只是想接着写下一个字节，等的是 `DRE`。**

### SYNCBUSY（0x1C）与 ADDR（0x24）

`SYNCBUSY` 三位：`SWRST`(bit0)、`ENABLE`(bit1)、`CTRLB`(bit2)。写完 `CTRLA.ENABLE` 等 bit1，写完 `CTRLB` 等 bit2。

`ADDR` 只对从机有意义：`ADDR[7:0]` 是本机地址，`ADDRMASK[7:0]` 是掩码或第二地址。主机用不到它。

## 8.4 USART 硬件：一根没有时钟线的串口

USART 和 SPI 最大的区别是：**没有时钟线**。接收方只能靠"双方事先约定的波特率"和每一帧开头的起始位重新对齐。名字里的 S 表示它也支持同步模式（多一根 XCK 时钟线），本章只用异步。

一帧的样子（8 位数据、无校验、1 位停止位，也就是常说的 8N1）：

```text
空闲(高) ─┐  起始位    D0 D1 D2 D3 D4 D5 D6 D7   停止位  ┌─ 空闲(高)
          └─── 0 ────┴──────────────────────────┴── 1 ──┘
```

| 部分 | 作用 | SAMD21 里由谁决定 |
|---|---|---|
| 起始位 | 把空闲的高电平拉低，接收方据此重新对齐自己的位时钟 | 硬件自动产生 |
| 5~9 位数据 | 真正的内容 | `CTRLB.CHSIZE`；低位先发还是高位先发由 `CTRLA.DORD` |
| 校验位（可选） | 奇偶校验 | `CTRLA.FORM=0x1` + `CTRLB.PMODE` |
| 停止位（1 或 2） | 拉回高电平，给接收方一个结束标志 | `CTRLB.SBMODE` |
| 波特率 | 每一位持续多久，双方必须一致 | `BAUD` |

起始位为什么能"重新对齐"：接收方平时盯着 RxD，看到高电平里出现一个下降沿，就知道新的一帧来了，于是从这一刻起用本地时钟数位。所以每一帧都会重新同步一次，**误差最多在一帧之内累积**。

### 8.4.1 为什么要 16 倍过采样和三取二

没有时钟线，接收方只能"猜"每一位的中心。SAMD21 的做法是每位采样 16 次，取中间三个点表决：

```text
起始位              每一位数据
  │  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16
  └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
                        └── 7、8、9 三点表决 ──┘
```

好处有两个：在位中心采样，对波特率误差最宽容；三取二表决可以滤掉单个采样点上的毛刺。`CTRLA.SAMPR` 选采样率（0x0=16 倍算术，复位默认；0x1=16 倍分数；0x2/0x3=8 倍；0x4=3 倍），`CTRLA.SAMPA` 选表决点的位置。**正常情况下这两个字段不要动。**

### 8.4.2 波特率怎么算（算术模式）

手册 Table 25-2，异步算术模式（`SAMPR=0`，每位 16 个采样点）：

```text
f_BAUD = (f_ref / 16) × (1 − BAUD / 65536)        BAUD = 65536 × (1 − 16 × f_BAUD / f_ref)
```

注意 `BAUD` 不是"分频比"，而是一个**从 65536 往下数的装载值**：BAUD 越大，位时间越长、波特率越低。16 位的宽度给了 1/65536 的分辨率，所以误差通常很小：

| 目标波特率（f_ref = 48 MHz） | BAUD 计算 | 实际波特率 | 误差 |
|---|---|---|---|
| 9600 | 65536 × (1 − 16×9600/48e6) = 65326 | 9613 | +0.14% |
| 115200 | 65536 × (1 − 16×115200/48e6) = 63019 | 115218 | +0.016% |
| 1000000 | 65536 × (1 − 16e6/48e6) = 43691 | 999985 | −0.0015% |

误差的定义是"实际与期望之差除以期望"。手册 Table 26-3 给出接收端的容限：8 位数据帧约 ±2.0%，9 位约 ±1.5%。**发收双方的误差会叠加，所以工程上一般要求两边都控制在 1% 以内**；像 9600 这种低波特率，一个 BAUD 步进（约 16 个 f_ref 周期）就已经占掉 0.14%，选时钟源时要把它算进去。

如果算术模式的误差不满足要求，可以把 `CTRLA.SAMPR` 设成 0x1 用**分数模式**：`BAUD` 拆成整数部分 `BAUD[12:0]` 和 1/8 小数部分 `FP[2:0]`，分辨率变成 1/(8×BAUD)。

```text
分数模式：f_BAUD = f_ref / (16 × (BAUD + FP/8))    验算：48 MHz、2 Mbaud → BAUD=1、FP=4
                                             48e6 / (16 × (1 + 4/8)) = 2e6，正好
```

## 8.5 使用逻辑：USART 初始化的顺序

```text
① 时钟：PM 开 APB 时钟 → GCLK 把 GCLK_SERCOMx_CORE 接到 GCLK0
② 引脚：PORT 里 PINCFG.PMUXEN=1 + PMUX 选功能 C
③ 模式：CTRLA 的 MODE=0x1（内部时钟）、CMODE=0（异步）、RXPO、TXPO、DORD
④ 帧格式：CTRLB 的 CHSIZE、SBMODE、（可选）PMODE
⑤ 波特率：BAUD
⑥ 使能：CTRLA.ENABLE=1，等 SYNCBUSY.ENABLE 清零
⑦ 收发器：CTRLB 的 TXEN、RXEN（写之前等 SYNCBUSY.CTRLB=0）
```

和 SPI 的关键差别：**USART 的发送器和接收器是分开开关的**（`TXEN` / `RXEN`），SPI 只有 `RXEN`（SPI 主机不存在"只收不发"，因为有收必有发）。

RxD 和 TxD 落在哪个 PAD 由 `CTRLA.TXPO`、`CTRLA.RXPO` 决定：

| `TXPO` | TxD | XCK | 备注 |
|---|---|---|---|
| 0x0 | PAD[0] | PAD[1] | 最常用 |
| 0x1 | PAD[2] | PAD[3] | 备选组合 |
| 0x2 | PAD[0] | 无 | PAD[2]=RTS、PAD[3]=CTS，硬件流控（本章不用） |

| `RXPO` | 0x0 | 0x1 | 0x2 | 0x3 |
|---|---|---|---|---|
| RxD 落在 | PAD[0] | PAD[1] | PAD[2] | PAD[3] |

常见的接法是"PAD[0] 发、PAD[1] 收"，也就是 `TXPO=0x0` + `RXPO=0x1`，占用的正好是同一个 SERCOM 相邻的两个引脚。

## 8.6 USART 寄存器逐个讲

### CTRLA（0x00）

| 位 | 名字 | 回答什么问题 |
|---|---|---|
| 4:2 | `MODE` | 0x1 = 用内部波特率发生器（异步串口的标准选择）；0x0 = 外部时钟 |
| 28 | `CMODE` | 0 = 异步（只有 TxD / RxD），1 = 同步（多一根 XCK） |
| 30 | `DORD` | 先发低位还是高位。**串口惯例是先发最低位，所以这里写 1（LSB）**，和 SPI 的默认刚好相反 |
| 27:24 | `FORM` | 帧格式：0x0 无校验，0x1 带奇偶校验，0x4/0x5 自动波特率（LIN） |
| 21:20 | `RXPO` | RxD 在哪个 PAD |
| 17:16 | `TXPO` | TxD（以及 RTS/CTS）在哪些 PAD |
| 15:13 | `SAMPR` | 采样率，默认 0x0（16 倍算术），别动 |
| 23:22 | `SAMPA` | 三个表决采样点的位置，别动 |
| 8 | `IBON` | 溢出报法：0 = 跟着数据流走（默认），1 = 立即报 |
| 1 / 0 | `ENABLE` / `SWRST` | 使能 / 软复位；`RUNSTDBY`(bit7) 决定 Standby 时是否继续接收 |

### CTRLB（0x04）

| 位 | 名字 | 回答什么问题 |
|---|---|---|
| 2:0 | `CHSIZE` | 字符长度：0x0=8 位（最常用），0x1=9 位，0x5~0x7=5~7 位 |
| 6 | `SBMODE` | 0 = 1 位停止位，1 = 2 位停止位 |
| 13 | `PMODE` | 校验方式：0 偶校验，1 奇校验（`FORM=0x1` 时才起作用） |
| 17 / 16 | `RXEN` / `TXEN` | 接收器 / 发送器开关 |

`CHSIZE` 和 `SBMODE` 是使能保护的（只能在 `ENABLE=0` 时写）；`TXEN` / `RXEN` 不是，可以在使能后随时改，但要等 `SYNCBUSY.CTRLB`。

### BAUD（0x0C，16 位）与 DATA（0x28，9 位有效）

算术模式下 `BAUD[15:0]` 就是 8.4.2 算出来的装载值；分数模式下高 3 位变成 `FP[2:0]`、低 13 位是 `BAUD[12:0]`。`BAUD` 是使能保护的，只能在 `ENABLE=0` 时写。

`DATA` 读写共用同一个地址：写 = 送发送缓冲（硬件自动补起始位、校验位、停止位），读 = 取接收缓冲。同样只允许在 `DRE=1` 时写、`RXC=1` 时读；9 位帧用 `DATA[8]`。

### INTFLAG（0x18）与 STATUS（0x1A）

| INTFLAG 位 | 名字 | 什么时候置 1 | 怎么清 |
|---|---|---|---|
| 0 | `DRE` | 发送缓冲空，可以写下一个字节 | 写 `DATA` |
| 1 | `TXC` | 整帧（含停止位）都发完了，而且没有新数据 | 写 1，或写新 `DATA` |
| 2 | `RXC` | 接收缓冲里有没读走的字节 | 读 `DATA` |
| 7 | `ERROR` | 有错，具体看 `STATUS` | 写 1 |
| 5 / 4 / 3 | `RXBRK` / `CTSIC` / `RXS` | 断帧（自动波特率）/ CTS 引脚变化 / 起始位检测 | 写 1 |

| STATUS 位 | 名字 | 含义 | 怎么办 |
|---|---|---|---|
| 0 | `PERR` | 奇偶校验错 | 丢弃这个字节，或让上层重发 |
| 1 | `FERR` | 帧错误：第一个停止位是 0，帧没有正常结束 | 多半是波特率不匹配；也可能是电平反相（RS-232）、线太长 |
| 2 | `BUFOVF` | 溢出：接收缓冲满了又来一个字节 | 读得太慢，降低波特率或改用中断/DMA（后面的章节） |
| 3 | `CTS` | 流控引脚当前电平（只读） | 没用流控就不用管 |

这三条错误信息有个必须记住的顺序：**先读 `STATUS`，再读 `DATA`**。手册明确写着 `FERR`、`BUFOVF`、`PERR` 描述的是"下一个将要读出的字符"；如果先把 `DATA` 读走，就再也对不上是哪一个字节出的错了。三个标志都是写 1 清零，关闭接收器（`RXEN=0`）也会自动清。

`TXC` 在这里的用处比 SPI 更实际：它表示最后一个字节真的从引脚上出去了。**切换 RS-485 收发方向、关闭发送器、进入低功耗之前，等的就是它。**

### SYNCBUSY（0x1C）

和 SPI 完全一样：`SWRST`(bit0)、`ENABLE`(bit1)、`CTRLB`(bit2)。使能后写 `CTRLB`（比如打开 `TXEN`/`RXEN`）之前，必须等 `SYNCBUSY.CTRLB=0`。

## 8.7 把引脚交给 SERCOM：PORT 的 PMUX

到 8.1.2 为止，我们只决定了信号落在哪个 PAD。PAD 是 SERCOM 内部的抽象接点，要真正连到 PA16 这样的引脚上，还得过 PORT 这一关。

每个引脚背后有一个 8 选 1 的开关：一边是 GPIO（`DIR`/`OUT`/`IN`），另一边是 8 个外设功能 A~H。

```text
                       ┌── A/B: EIC 外部中断、ADC/AC（模拟）
  引脚 ←─8 选 1 开关──┤   C: SERCOM   ← 本章用这个
（GPIO 或某个外设）     └── D~H: SERCOM-ALT / TC / TCC / GCLK_IO …
```

两件事必须都做，缺一不可：`PINCFGn.PMUXEN=1` 打开外设通路；`PMUXn.PMUXE/PMUXO` 选出功能号（A=0、B=1、C=2、D=3……H=7）。

| 寄存器 | 地址规则 | 说明 |
|---|---|---|
| `PINCFG[n]` | `0x40 + n` | bit0 `PMUXEN`；bit1 `INEN`、bit2 `PULLEN`、bit6 `DRVSTR` 对外设引脚仍然有效 |
| `PMUX[n/2]` | `0x30 + n/2` | 低 4 位 `PMUXE` 管偶数脚 `2n`，高 4 位 `PMUXO` 管奇数脚 `2n+1` |

给 PA16、PA17 接上 SERCOM1，只要三行（PA16/PA17 属于第 8 组：`16/2 = 8`）：

```c
/* PMUX[8] 的低 4 位管 PA16，高 4 位管 PA17；0x2 = 功能 C = SERCOM */
PORT_REGS->GROUP[0].PORT_PMUX[8] = PORT_PMUX_PMUXE_C | PORT_PMUX_PMUXO_C;

/* 打开外设通路：从这一刻起，这两个引脚的方向和输出值由 SERCOM 决定 */
PORT_REGS->GROUP[0].PORT_PINCFG[16] = PORT_PINCFG_PMUXEN_Msk;
PORT_REGS->GROUP[0].PORT_PINCFG[17] = PORT_PINCFG_PMUXEN_Msk;
```

三个容易踩的点：

1. **方向不用你写。** 手册在 `PINCFG.PMUXEN` 的位描述里说得很清楚：PMUXEN=1 后，"被选中的外设功能控制引脚的方向和输出值"，`DIR`/`OUT` 不再生效。给 SERCOM 的输出脚补一句 `DIRSET` 是多余的。
2. **`INEN` 和 SERCOM 收发无关。** 外设是直接从 PAD 取信号的，`INEN` 只决定你还能不能用 `IN` 读回这个引脚的电平（调试时有用，正常不用写）。
3. **`PULLEN`、`DRVSTR` 仍然归你。** 但 SERCOM 的输入脚上，内部上下拉**只能当下拉用**（手册 26.5.1：无法使能上拉）。RxD 悬空时打开下拉，能避免收到一串乱码。

哪些引脚能当 SERCOM 用，由手册 Table 7-1 的功能列决定。C 列是"主功能"（SERCOM），D 列是 SERCOM-ALT（同一个 SERCOM 的第二组引脚）：

| SERCOM | PAD[0] | PAD[1] | PAD[2] | PAD[3] | 功能列 |
|---|---|---|---|---|---|
| SERCOM0 | PA04 | PA05 | PA06 | PA07 | C |
| SERCOM0 | PA08 | PA09 | PA10 | PA11 | C（同一组也是 SERCOM2，在 D 列） |
| SERCOM1 | PA16 | PA17 | PA18 | PA19 | C（D 列上是 SERCOM3） |
| SERCOM2 | PA12 | PA13 | PA14 | PA15 | C（D 列上是 SERCOM4） |
| SERCOM3 | PA22 | PA23 | PA24 | PA25 | C（D 列上是 SERCOM5） |
| SERCOM4 | PB08 | PB09 | PB10 | PB11 | C |
| SERCOM5 | PB16 | PB17 | PA20 | PA21 | C |

注意 PA24/PA25 同时是 USB 的 D−/D+，PA00/PA01 是 32.768 kHz 晶振脚，PB08/PB09 等在 VDDANA 域——选引脚前先把第 4 章那张"特殊引脚表"过一遍。上表按 48 脚 SAMD21G 的封装整理，用任何一组之前回查手册 Table 7-1 的 C、D 两列确认功能号和 PAD 号。

## 8.8 例程一：SPI 主机读 SPI Flash 的 ID

**硬件需求**：一块 SPI Flash（以 W25Q 系列为例）接在 SERCOM1 上：PA16=MOSI、PA17=SCK、PA18=CS（普通 GPIO）、PA19=MISO；Flash 供电 3.3 V，WP#、HOLD# 拉高。

**要发的命令**：`0x9F`（JEDEC ID）。它会连续回 3 个字节：厂商 ID、存储类型、容量。W25Q32 常见回值 `EF 40 16`（厂商 0xEF = Winbond）——**具体数值以你手上 Flash 的手册为准**，这里只用来判断"读回来的东西像不像话"。

**从需求推到寄存器**：

```text
要 4 根线 → SERCOM1 的 PAD0/PAD1/PAD3 → DOPO=0x0（DO=PAD0、SCK=PAD1）、DIPO=0x3（DI=PAD3）
CS 想自己控制 → CTRLB.MSSEN=0，PA18 当普通 GPIO
要收数据 → CTRLB.RXEN=1
Flash 支持模式 0 → CPOL=0、CPHA=0、DORD=MSB
48 MHz 想要约 4 MHz → BAUD=5
```

```c
#include "sam.h"

#define FLASH_CS_PIN   18
#define FLASH_CS_MASK  (1UL << FLASH_CS_PIN)

static void spi_init(void)
{
    /* ① SERCOM1 在桥 C：APB 时钟复位后是关的，先开它，再把 48 MHz 的 GCLK0 接过去 */
    PM_REGS->PM_APBCMASK |= PM_APBCMASK_SERCOM1_Msk;
    GCLK_REGS->GCLK_CLKCTRL = GCLK_CLKCTRL_ID_SERCOM1_CORE
                            | GCLK_CLKCTRL_GEN_GCLK0
                            | GCLK_CLKCTRL_CLKEN_Msk;
    while (GCLK_REGS->GCLK_STATUS & GCLK_STATUS_SYNCBUSY_Msk) { }

    /* ② 引脚交给 SERCOM1：PA16=PAD0(MOSI)、PA17=PAD1(SCK)、PA19=PAD3(MISO) */
    PORT_REGS->GROUP[0].PORT_PMUX[8] = PORT_PMUX_PMUXE_C | PORT_PMUX_PMUXO_C;
    PORT_REGS->GROUP[0].PORT_PMUX[9] = PORT_PMUX_PMUXO_C;      /* 只给 PA19，PMUXE 留给 PA18 */
    PORT_REGS->GROUP[0].PORT_PINCFG[16] = PORT_PINCFG_PMUXEN_Msk;
    PORT_REGS->GROUP[0].PORT_PINCFG[17] = PORT_PINCFG_PMUXEN_Msk;
    PORT_REGS->GROUP[0].PORT_PINCFG[19] = PORT_PINCFG_PMUXEN_Msk;

    /* ③ CS 用普通 GPIO。先准备高电平再打开输出，避免上电瞬间误选中 Flash */
    PORT_REGS->GROUP[0].PORT_OUTSET = FLASH_CS_MASK;
    PORT_REGS->GROUP[0].PORT_DIRSET = FLASH_CS_MASK;

    /* ④ 模式与帧格式。这些位只能在 ENABLE=0 时写 */
    SERCOM1_REGS->SPIM.SERCOM_CTRLA =
          SERCOM_SPIM_CTRLA_MODE_SPI_MASTER
        | SERCOM_SPIM_CTRLA_DOPO_PAD0          /* MOSI=PAD0、SCK=PAD1 */
        | SERCOM_SPIM_CTRLA_DIPO_PAD3          /* MISO=PAD3 */
        | SERCOM_SPIM_CTRLA_DORD_MSB
        | SERCOM_SPIM_CTRLA_CPOL_IDLE_LOW
        | SERCOM_SPIM_CTRLA_CPHA_LEADING_EDGE; /* 模式 0：上升沿采样 */

    /* CHSIZE=8 位；RXEN=1，否则收进来的字节直接丢掉 */
    SERCOM1_REGS->SPIM.SERCOM_CTRLB = SERCOM_SPIM_CTRLB_CHSIZE_8_BIT
                                    | SERCOM_SPIM_CTRLB_RXEN_Msk;

    /* ⑤ f_SCK = 48 MHz / (2×5+1) ≈ 4.36 MHz */
    SERCOM1_REGS->SPIM.SERCOM_BAUD = SERCOM_SPIM_BAUD_BAUD(5);

    /* ⑥ 使能。CTRLA 写同步，等 SYNCBUSY.ENABLE 清零后再往下走 */
    SERCOM1_REGS->SPIM.SERCOM_CTRLA |= SERCOM_SPIM_CTRLA_ENABLE_Msk;
    while (SERCOM1_REGS->SPIM.SERCOM_SYNCBUSY & SERCOM_SPIM_SYNCBUSY_ENABLE_Msk) { }
}

/* SPI 是全双工：写一个字节的动作同时收一个字节。参数是"我要发什么"，返回值是"同时收到了什么" */
static uint8_t spi_transfer(uint8_t tx)
{
    /* DRE=1 才表示发送缓冲空，这时候写 DATA 才会被接受 */
    while ((SERCOM1_REGS->SPIM.SERCOM_INTFLAG & SERCOM_SPIM_INTFLAG_DRE_Msk) == 0) { }
    SERCOM1_REGS->SPIM.SERCOM_DATA = tx;      /* 写 DATA：字节进移位寄存器，SCK 随即开始跑 */

    /* RXC=1 表示移位寄存器已经把 8 位收满并搬进接收缓冲 */
    while ((SERCOM1_REGS->SPIM.SERCOM_INTFLAG & SERCOM_SPIM_INTFLAG_RXC_Msk) == 0) { }
    return (uint8_t)SERCOM1_REGS->SPIM.SERCOM_DATA;   /* 读 DATA 会顺带清 RXC */
}

static void flash_read_id(uint8_t *id3)
{
    /* 整个事务期间 CS 必须一直保持低：Flash 靠它判断命令、地址、数据属于同一次操作 */
    PORT_REGS->GROUP[0].PORT_OUTCLR = FLASH_CS_MASK;

    (void)spi_transfer(0x9F);        /* 第 1 次写：把命令发出去，同时收到的字节无意义，丢掉 */
    id3[0] = spi_transfer(0xFF);     /* 之后每写一个 0xFF（凑时钟用），才能收回一个字节 */
    id3[1] = spi_transfer(0xFF);
    id3[2] = spi_transfer(0xFF);

    PORT_REGS->GROUP[0].PORT_OUTSET = FLASH_CS_MASK;   /* CS 拉高，事务结束 */
}

int main(void)
{
    uint8_t id[3];

    spi_init();
    flash_read_id(id);      /* 调试时在这一行打断点，看 id[0..2] 是不是 EF 40 16 */
    while (1) { }
}
```

这段代码里有三个"为什么"，都来自硬件而不是语法：

| 写法 | 硬件原因 |
|---|---|
| 一次事务里 `spi_transfer` 写了 4 次 | 命令 1 次 + 读回 3 个字节各 1 次。移位寄存器每转一圈必然一进一出，**不给时钟就收不到数据**，所以后面三次必须写"哑元"（0xFF 只是习惯） |
| CS 在函数外拉低、函数内不动 | CS 属于事务边界。在字节之间抖动 CS，Flash 会认为事务结束，后面的字节就成了新命令 |
| 第 1 次 `spi_transfer` 的返回值丢掉 | 那一个字节是发命令的同时从 MISO 上收进来的东西，此时 Flash 还没开始回答 |

## 8.9 例程二：USART 回环（收到什么发回什么）

**硬件需求**：USB 转串口模块接在 SERCOM0 上：PA08=TxD（接模块的 RX）、PA09=RxD（接模块的 TX）、GND 共地；串口终端设成 115200 8N1。

```text
MCU PA08 (TxD) ──→ 模块 RXD
MCU PA09 (RxD) ←── 模块 TXD
MCU GND        ──── 模块 GND   （一定要共地，否则收到的全是乱码）
```

**从需求推到寄存器**：

```text
两根线 → SERCOM0 的 PAD0/PAD1 → TXPO=0x0（TxD=PAD0）、RXPO=0x1（RxD=PAD1）
异步、无时钟线 → CTRLA.MODE=0x1（内部时钟）、CMODE=0
串口惯例先发最低位 → DORD=LSB（写 1）
8N1 → CHSIZE=8 位、SBMODE=1 位停止位、FORM=0（无校验）
115200 @ 48 MHz → BAUD = 65536 × (1 − 16×115200/48e6) = 63019
```

```c
#include "sam.h"

static void uart_init(void)
{
    /* ① SERCOM0 也在桥 C：先开 APB 时钟，再把 GCLK0 接到 SERCOM0_CORE */
    PM_REGS->PM_APBCMASK |= PM_APBCMASK_SERCOM0_Msk;
    GCLK_REGS->GCLK_CLKCTRL = GCLK_CLKCTRL_ID_SERCOM0_CORE
                            | GCLK_CLKCTRL_GEN_GCLK0
                            | GCLK_CLKCTRL_CLKEN_Msk;
    while (GCLK_REGS->GCLK_STATUS & GCLK_STATUS_SYNCBUSY_Msk) { }

    /* ② PA08/PA09 交给 SERCOM0（功能 C）。谁是 TxD、谁是 RxD 由 CTRLA 决定，不由引脚决定 */
    PORT_REGS->GROUP[0].PORT_PMUX[4] = PORT_PMUX_PMUXE_C | PORT_PMUX_PMUXO_C;
    PORT_REGS->GROUP[0].PORT_PINCFG[8] = PORT_PINCFG_PMUXEN_Msk;
    PORT_REGS->GROUP[0].PORT_PINCFG[9] = PORT_PINCFG_PMUXEN_Msk;

    /* ③ 模式：内部波特率 + 异步 + 低位先发；TxD 放 PAD[0]、RxD 放 PAD[1] */
    SERCOM0_REGS->USART_INT.SERCOM_CTRLA =
          SERCOM_USART_INT_CTRLA_MODE_USART_INT_CLK
        | SERCOM_USART_INT_CTRLA_CMODE_ASYNC
        | SERCOM_USART_INT_CTRLA_DORD_LSB
        | SERCOM_USART_INT_CTRLA_TXPO_PAD0
        | SERCOM_USART_INT_CTRLA_RXPO_PAD1;

    /* ④ 帧格式：8 位数据、1 位停止位、无校验（FORM 的复位值就是 0） */
    SERCOM0_REGS->USART_INT.SERCOM_CTRLB = SERCOM_USART_INT_CTRLB_CHSIZE_8_BIT;

    /* ⑤ 波特率：装载值越小波特率越高 */
    SERCOM0_REGS->USART_INT.SERCOM_BAUD = SERCOM_USART_INT_BAUD_BAUD(63019);

    /* ⑥ 使能，先等 SYNCBUSY.ENABLE */
    SERCOM0_REGS->USART_INT.SERCOM_CTRLA |= SERCOM_USART_INT_CTRLA_ENABLE_Msk;
    while (SERCOM0_REGS->USART_INT.SERCOM_SYNCBUSY & SERCOM_USART_INT_SYNCBUSY_ENABLE_Msk) { }

    /* ⑦ 打开收发器。写 CTRLB 要等 SYNCBUSY.CTRLB，否则会产生 APB 错误 */
    SERCOM0_REGS->USART_INT.SERCOM_CTRLB |= SERCOM_USART_INT_CTRLB_TXEN_Msk
                                          | SERCOM_USART_INT_CTRLB_RXEN_Msk;
    while (SERCOM0_REGS->USART_INT.SERCOM_SYNCBUSY & SERCOM_USART_INT_SYNCBUSY_CTRLB_Msk) { }
}

/* 发一个字节：DRE=1 表示发送缓冲空，写进去后硬件自己补起始位和停止位 */
static void uart_putc(uint8_t b)
{
    while ((SERCOM0_REGS->USART_INT.SERCOM_INTFLAG & SERCOM_USART_INT_INTFLAG_DRE_Msk) == 0) { }
    SERCOM0_REGS->USART_INT.SERCOM_DATA = b;
}

/* 收一个字节：先读 STATUS 才知道这一帧有没有错；返回 -1 表示有错 */
static int uart_getc(void)
{
    uint16_t st = SERCOM0_REGS->USART_INT.SERCOM_STATUS;
    uint8_t  d  = (uint8_t)SERCOM0_REGS->USART_INT.SERCOM_DATA;   /* 读 DATA 顺带清 RXC */

    if (st & (SERCOM_USART_INT_STATUS_FERR_Msk | SERCOM_USART_INT_STATUS_BUFOVF_Msk))
    {
        /* 错误标志写 1 清零；数据已经读走，接收缓冲不会被堵住 */
        SERCOM0_REGS->USART_INT.SERCOM_STATUS = SERCOM_USART_INT_STATUS_FERR_Msk
                                              | SERCOM_USART_INT_STATUS_BUFOVF_Msk;
        return -1;
    }
    return (int)d;
}

int main(void)
{
    uart_init();
    while (1)
    {
        /* RXC=1 说明接收缓冲里有货。这里用轮询演示，实际工程应改用中断或 DMA */
        if (SERCOM0_REGS->USART_INT.SERCOM_INTFLAG & SERCOM_USART_INT_INTFLAG_RXC_Msk)
        {
            int c = uart_getc();
            if (c >= 0)
            {
                uart_putc((uint8_t)c);      /* 回环：收到什么就原样发回去；c<0 的错帧直接丢掉 */
            }
        }
    }
}
```

几个值得注意的地方：轮询写法在 115200 下勉强够用（一个字节约 87 μs），但主循环里一旦有别的活就会溢出；出错时**必须把 `DATA` 读走**，否则接收缓冲一直是满的，后面每一帧都是 `BUFOVF`。想验证"芯片自己是好的、问题在外部接线或对方设备"，可以把 `TXPO` 和 `RXPO` 指到同一个 PAD 做内部回环——手册 26.6.3.6 说这种回环是经过焊盘的，示波器上还能看到波形。

## 8.10 初始化清单

**SPI 主机**：时钟（APB + GCLK_CORE）→ 引脚（`PMUXEN` + 功能 C；CS 用 GPIO 时先 `OUTSET` 再 `DIRSET`）→ `CTRLA`（MODE=0x3、DOPO、DIPO、CPOL/CPHA、DORD）→ `CTRLB`（CHSIZE、RXEN）→ `BAUD` → `ENABLE` 并等 `SYNCBUSY.ENABLE`。

**SPI 从机**：同上去掉 `BAUD`，`MODE=0x2`；要发确定的首字节就打开 `PLOADEN` 并在 SS 为高时先写一个字节；要用地址匹配就配 `FORM=0x2` + `ADDR` + `AMODE`。

**USART**：时钟 → 引脚 → `CTRLA`（MODE=0x1、CMODE=0、RXPO、TXPO、DORD=1、可选 FORM）→ `CTRLB`（CHSIZE、SBMODE、可选 PMODE）→ `BAUD` → `ENABLE` 并等 `SYNCBUSY.ENABLE` → `TXEN`/`RXEN` 并等 `SYNCBUSY.CTRLB`。

初始化之后，运行中真正会做的事只有三件：**等 `DRE` 写 `DATA`、等 `RXC` 读 `DATA`、出错时读 `STATUS` 并写 1 清标志**。其余寄存器都属于"初始化时定一次"。

## 8.11 一张图记住整章

```text
          SERCOMx（先定模式：USART / SPI 主机 / SPI 从机）
                         │
  ┌───────────┬──────────┴──────────┬───────────────────────┐
 帧格式/长度   引脚落点             位时钟                 数据搬运
 CTRLA/CTRLB  CTRLA.RXPO/TXPO      BAUD           DATA + INTFLAG(DRE/RXC/TXC)
 CTRLA.DORD   CTRLA.DOPO/DIPO    （从机不用）       错误看 STATUS；引脚走 PMUX 功能 C
```

## 8.12 常见问题

| 现象 | 原因 | 检查什么 |
|---|---|---|
| 引脚上一点波形都没有 | 引脚还归 GPIO，或者 SERCOM 没有时钟 | `PINCFG.PMUXEN`、`PMUX` 是否功能 C、`GCLK_CLKCTRL`、`PM_APBCMASK` |
| SPI 读回来全是 0xFF 或 0x00 | CPOL/CPHA 与从机不一致；或者 MISO 没落在 `DIPO` 指定的 PAD 上 | 从机手册的时钟模式；`DIPO` 与原理图引脚 |
| SPI 只有第一个字节是对的 | CS 在字节之间被拉高，事务被拆断 | CS 必须在整个事务期间保持低，且不能让别的代码碰它 |
| SPI 主机永远等不到 `RXC` | 没开 `CTRLB.RXEN`，接收器关着，数据直接被丢 | `RXEN` |
| USART 收到的全是乱码 | 波特率不一致或误差过大，或者 `DORD` 写成了 MSB | 两边波特率、`BAUD` 计算、`CTRLA.DORD`（应为 LSB） |
| USART 完全没反应 | TxD/RxD 接反，或者没有共地 | 线序、GND、`TXPO`/`RXPO` 与引脚是否对应 |
| `STATUS.FERR` 反复出现 | 停止位不是高电平 | 波特率；RS-232 电平是否反相（需要 MAX3232 这类转换）；线是否过长 |
| `STATUS.BUFOVF` 反复出现 | 读得太慢，2 级接收缓冲溢出 | 降低波特率；改用中断或 DMA（后面章节） |
| 第一个字节丢失 | 发送端没等 `DRE` 就写；或接收端使能瞬间的起始沿被吞掉 | 写 `DATA` 前确认 `DRE=1`；接收端先读一次 `DATA` 清空 |
| 写完 `CTRLB` 就 HardFault 或复位 | `SYNCBUSY.CTRLB` 没清零就写了 | 每次写 `CTRLB` 前等 `SYNCBUSY.CTRLB=0` |

## 8.13 寄存器速查

| 偏移 | 寄存器 | SPI 主机 / 从机 | USART |
|---|---|---|---|
| 0x00 | `CTRLA` | MODE(0x3/0x2)、DORD、CPOL、CPHA、DIPO、DOPO、FORM、IBON、ENABLE、SWRST | MODE(0x1)、CMODE、DORD、FORM、RXPO、TXPO、SAMPR、SAMPA、IBON、RUNSTDBY、ENABLE、SWRST |
| 0x04 | `CTRLB` | CHSIZE、RXEN、MSSEN、PLOADEN、AMODE、SSDE | CHSIZE、SBMODE、PMODE、TXEN、RXEN |
| 0x0C | `BAUD` | 8 位：f_SCK = f_ref/(2×BAUD+1) | 16 位：算术装载值；分数模式为 FP[2:0] + BAUD[12:0] |
| 0x14 / 0x16 | `INTENCLR` / `INTENSET` | DRE、TXC、RXC、SSL、ERROR | DRE、TXC、RXC、RXS、CTSIC、RXBRK、ERROR |
| 0x18 | `INTFLAG` | 同上（DRE=0、TXC=1、RXC=2、SSL=3、ERROR=7） | 同上 |
| 0x1A | `STATUS` | BUFOVF(bit2) | PERR(0)、FERR(1)、BUFOVF(2)、CTS(3)、ISF(4)、COLL(5) |
| 0x1C | `SYNCBUSY` | SWRST(0)、ENABLE(1)、CTRLB(2) | 同左 |
| 0x24 | `ADDR` | 从机地址 / 掩码 | 无 |
| 0x28 | `DATA` | 写=发送缓冲，读=接收缓冲，9 位用 DATA[8] | 同左 |

SERCOM0 基址 `0x4200_0800`，往后每个实例加 `0x400`；GCLK 的 `SERCOMx_CORE` ID 从 0x14 递增（SLOW 时钟只有 I2C 需要）。另外两个本章没细讲的寄存器：`RXPL`(0x0E) 只在 IrDA 模式用，`DBGCTRL`(0x30) 决定调试暂停时波特率发生器是否停。

## 8.14 最重要的几句话

1. SERCOM 是一个引擎三种模式，`CTRLA.MODE` 一选定下来，整个实例的资源就都归它。
2. SPI 的四根线要经过两次决定：先在 SERCOM 内部落到 PAD（`DOPO`/`DIPO`），再由 PORT 的 PMUX 落到真实引脚。
3. SPI 全双工：写一个字节才能收一个字节；`DRE` 说"能写"，`RXC` 说"能读"，`TXC` 说"真的发完了"。
4. USART 没有时钟线，靠起始位重新对齐、靠约定波特率数位；`BAUD` 是从 65536 往下数的装载值，不是分频比。
5. 波特率误差要两边一起算，工程上各控制在 1% 以内；手册给的接收容限是 8 位帧 ±2.0%。
6. USART 的 `FERR`/`BUFOVF`/`PERR` 必须在读 `DATA` **之前**读 `STATUS`，否则不知道是哪一帧出的错。
7. 引脚交给 SERCOM = `PINCFG.PMUXEN=1` + `PMUX` 选功能 C；之后方向由外设接管，但 `PULLEN`/`DRVSTR` 仍归你（SERCOM 输入脚只能下拉）。
8. `CTRLA`/`CTRLB`/`BAUD` 是使能保护的；`ENABLE` 和 `CTRLB` 写完要等 `SYNCBUSY`，否则写入被丢弃甚至报 APB 错误。

## 8.15 自测

1. PA16 要作为 SERCOM1 的 MOSI，PORT 里至少写哪两个寄存器？为什么方向寄存器不用写？
2. SPI 主机要读 Flash 的 3 个字节，为什么一共写了 4 次 `DATA`？第 1 次写的 `0x9F` 到哪去了？
3. `INTFLAG.DRE` 和 `INTFLAG.TXC` 都在讲发送，区别是什么？什么场合必须等 `TXC`？
4. f_ref = 48 MHz、要 115200 波特率，`BAUD` 写多少？如果对方比你快 1%，会发生什么？
5. 收到一帧 `STATUS.FERR=1`，为什么必须先把 `DATA` 读走，再写 1 清标志？不读 `DATA` 会怎样？
6. 写完 `CTRLA.ENABLE=1` 立刻去写 `CTRLB.TXEN=1`，为什么要先等 `SYNCBUSY.CTRLB=0`？

## 附录：以后查手册怎么找 SPI 与 USART

查的顺序是固定的，不必通读：

```text
① SERCOM 总章（第 25 章）：模式表、波特率公式 Table 25-2、同步与使能规则
② 模式章：USART 是第 26 章，SPI 是第 27 章（I2C 是第 28 章）
③ 章内的 Register Summary：一张表列出所有偏移
④ 单个寄存器的位描述：Enable-Protected 与 Write-Synchronized 这两个属性最重要
⑤ 引脚：回到第 7 章 Table 7-1，看功能 C / D 两列的 SERCOM 与 PAD 号
```

| 想查什么 | 手册位置 |
|---|---|
| 波特率公式与误差 | 25.6.2.3 与 Table 25-2；接收容限 Table 26-3 |
| 引脚方向由谁控制、PULLEN 的限制 | 26.5.1（USART）、27.5.1（SPI） |
| 溢出错误的两种报法 | `CTRLA.IBON` 的位描述；SPI 另见 27.6.2.7 |
| 从机预装与地址匹配 | 27.6.3.1、27.6.3.2 |
| 内部回环怎么接 | 26.6.3.6 |

中断、DMA、睡眠唤醒（`RXS`、`SSL`、`SSDE`、`RUNSTDBY`）属于后面的章节：本章只用轮询，把硬件流程走通。
