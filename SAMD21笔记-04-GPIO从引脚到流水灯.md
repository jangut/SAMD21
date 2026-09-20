
# 第 4 章 GPIO：从一个引脚到流水灯

> **芯片**：ATSAMD21G18A
> **本章目标**：理解 GPIO 的硬件原理，知道一个引脚为什么要配成输入或输出，以及这些配置在 SAMD21 上对应哪些寄存器；最后用 LED 和按键做出一个流水灯。
> **学习顺序**：硬件是什么 → GPIO 能做什么 → 为什么需要输入缓冲和上下拉 → 软件要配置什么 → 寄存器 → LED → 按键 → 流水灯。

## 4.0 GPIO 是什么

GPIO = General Purpose Input/Output（通用输入/输出）。一句话：**GPIO 是 MCU 用来观察外部电平、或者控制外部电平的引脚**。

| 行为 | 做什么 | 典型用途 |
|---|---|---|
| 输出 | 主动给出高/低电平 | LED、继电器、蜂鸣器 |
| 输入 | 读取外部给的电平 | 按键、传感器的数字输出 |

所以学 GPIO 就是回答两个问题：**这个引脚要不要主动控制外部电路？如果不要，我怎么知道外部给了它什么电平？**

## 4.1 硬件：引脚里面是两个开关

引脚内部的输出级叫**推挽输出（Push-Pull）**：上管接 VDD，下管接 GND。

```text
VDD ──[上管]──┬── 引脚
GND ──[下管]──┘
```

| 上管 | 下管 | 引脚被接到 | 结果 |
|---|---|---|---|
| 开 | 关 | VDD | 输出高 |
| 关 | 开 | GND | 输出低 |
| 关 | 关 | 悬空 | 高阻（三态） |

所以：**"输出高"的本质是把引脚主动拉向 VDD，"输出低"是拉向 GND**。而三态意味着引脚既不拉高也不拉低，可以被外部电路驱动——这是后面 I2C 总线要用的状态，本章的 LED 用不到。

由此还有一条硬件常识：**两个输出引脚不能直接对接**。一个输出高、一个输出低，就是 VDD 经两个开关直通 GND，会烧。要共享一根线，必须让它们处于三态。

## 4.2 读外部信号时，为什么不能保持输出

假设引脚一直输出高，按键另一端接 GND：

```text
VDD ──[ 引脚输出高 ]── 按键 ── GND      ← 按下即 VDD 直通 GND
```

按下按键就等于把 VDD 和 GND 通过引脚短接，电流从引脚灌进地。所以**读外部信号之前必须先关掉输出驱动**，这就是输入方向：引脚不再主动拉高拉低，只是"观察"引脚上的电压。

## 4.3 输入在读什么：从电压到 0/1

引脚上看的是**实际电压**：接近 VDD 判为 1，接近 GND 判为 0。这个电压要送进 MCU 内部逻辑，需要一条通路：

```text
引脚 ──▶ 输入缓冲 ──▶ 内部数字逻辑 ──▶ IN 寄存器
```

这条通路由 `PINCFG.INEN` 控制，而且**复位后默认是关闭的**。于是两个概念必须分开：

| 概念 | 含义 | 由谁控制 |
|---|---|---|
| 引脚用作输入 | 关掉输出驱动，引脚不再主动驱动 | `DIRCLR`（DIR 位 = 0） |
| 输入缓冲打开 | 允许电平进入 MCU 内部，`IN` 才读得到 | `PINCFG.INEN = 1` |

只配了输入方向、没开 `INEN`，读 `IN` 永远是 0。这是新手第一个坑。

## 4.4 为什么输入还需要上拉/下拉

按键一端接引脚、一端接 GND：按下时引脚被拉到 0，没问题；**松开时引脚和谁都不连，这就是"悬空"**。输入引脚阻抗很高，悬空时电压没有确定来源，读数会乱跳。

解决办法是在引脚和 VDD（或 GND）之间接一个大电阻：

| 接法 | 松开时 | 按下时（按键接 GND） |
|---|---|---|
| 上拉（引脚—R—VDD） | 1 | 0 |
| 下拉（引脚—R—GND） | 0 | 外部拉高时为 1 |

```text
        VDD                    GPIO
         │                      │
        [R]                    [R]
         │                      │
GPIO ────┤                      ├──── GPIO
         │                      │
       按键                    按键        ← 下拉的按键应接 VDD
         │                      │
        GND                    GND
        上拉：默认 1            下拉：默认 0
```

电阻不能省：没有电阻就是 VDD 经按键直通 GND。**上拉不是"强行把引脚变成 1"，而是"没人拉低时它默认是 1"。**


## 4.5 SAMD21 的特殊之处：上下拉方向由 OUT 决定

大部分单片机有"上拉""下拉"两个独立开关，SAMD21 不是：它只有一个 `PINCFG.PULLEN` 负责**打开电阻**，至于是上拉还是下拉，由**输出值 `OUT`** 决定。

| PULLEN | OUT | 结果 |
|---|---|---|
| 1 | 1 | 内部上拉 |
| 1 | 0 | 内部下拉 |
| 0 | — | 不接上下拉（悬空） |

所以"输入 + 内部上拉"要同时满足四件事：**输入方向（`DIRCLR`）＋ 输入缓冲（`INEN`）＋ 上下拉使能（`PULLEN`）＋ `OUT = 1`**。少最后一条，得到的就是下拉——这是第二个坑。

到这里，一个 GPIO 的全部属性可以列成一张表，后面的代码都只是它的翻译：

| 属性 | 作用 | 什么时候改 |
|---|---|---|
| 方向 | 输入还是输出 | 初始化定，运行中一般不动 |
| 输出电平 | 输出时是高还是低 | 运行中随时改（三个动作） |
| 输入缓冲 | 是否允许 MCU 读取引脚 | 初始化定 |
| 上下拉 | 悬空时提供默认电平 | 初始化定 |
| 驱动强度 | 输出能力大小 | 初始化定（LED 那节再说） |
| 复用 | 这个脚归 GPIO 还是外设 | 初始化定，本章不涉及 |

## 4.6 把硬件概念翻译成寄存器

现在才看寄存器。每个寄存器都应该能回答一个明确的问题：

| 我要做什么 | SAMD21 的配置 |
|---|---|
| 配成输出 / 输入 | `DIRSET` / `DIRCLR` |
| 输出高 / 低 / 翻转 | `OUTSET` / `OUTCLR` / `OUTTGL` |
| 读取引脚 | `IN` |
| 打开输入缓冲 | `PINCFG.INEN` |
| 打开上下拉 / 决定方向 | `PINCFG.PULLEN` / `OUT` |
| 增强驱动 | `PINCFG.DRVSTR` |
| 交给外设 | `PMUXEN + PMUX`（后续章节） |

**寄存器不是新知识，它只是硬件的控制开关。** 例如"让 PA05 输出低电平"这件事：

```text
我要让 PA05 输出低
      ↓ 硬件上
关闭输入用途 → 让输出驱动工作 → 让下管导通
      ↓ SAMD21 上
DIRSET  →  OUTCLR
```

寄存器按**引脚组**组织，每组一套完整寄存器，代码里就是 `Group[0]` / `Group[1]`：

| 组       | 覆盖引脚      | 代码                    | 基址            |
| ------- | --------- | --------------------- | ------------- |
| Group 0 | PA00–PA31 | `PORT_REGS->GROUP[0]` | `0x4100_4400` |
| Group 1 | PB00–PB31 | `PORT_REGS->GROUP[1]` | `0x4100_4480` |

PA05 和 PB05 都是"第 5 位"，区别只在 Group：

```c
PORT_REGS->GROUP[0].PORT_DIRSET = (1UL << 5);   /* PA05 */
PORT_REGS->GROUP[1].PORT_DIRSET = (1UL << 5);   /* PB05 */
```

组写错时，位号完全正确，但操作的是另一个物理引脚——这是第三个坑。
注：PORT 的 64 个位置（PA00–PA31、PB00–PB31）是硅片最大规格，用来覆盖 E/G/J 三种封装；本器件 48 脚只引出 38 个 I/O，其余位置在寄存器里存在、外面却没有脚。判断某个引脚存不存在，查手册 Table 7-1 的脚号列。

## 4.7 逐个看本章用到的寄存器

### DIR：方向

`DIR` = Direction，`0` 是输入、`1` 是输出。SAMD21 给了两个专用寄存器，不用去读-改-写整个 DIR：

```c
PORT_REGS->GROUP[0].PORT_DIRSET = (1UL << 5);   /* 把 PA05 设成输出，其他位不动 */
PORT_REGS->GROUP[0].PORT_DIRCLR = (1UL << 5);   /* 把 PA05 设成输入 */
```

其中 `1UL << 5` 是二进制 `0010 0000`：第 5 位为 1，表示"只对第 5 个引脚下手"。

### OUT：电平，和三个动作

`OUT` 决定输出电平（1 高、0 低），并且兼职决定上下拉方向（4.5）。改电平有三个"动作寄存器"：

| 寄存器 | 动作 | 语义 |
|---|---|---|
| `OUTSET` | 输出高 | 掩码里为 1 的脚拉高，其余不动 |
| `OUTCLR` | 输出低 | 掩码里为 1 的脚拉低，其余不动 |
| `OUTTGL` | 翻转 | 掩码里为 1 的脚取反，其余不动 |

### 为什么用 OUTSET，而不是直接改 OUT？

新手常见写法是 `PORT_REGS->GROUP[0].PORT_OUT |= (1UL << 5)`。它也能跑，但它其实是三步：

```text
读 OUT  →  改其中一位  →  把整个 OUT 写回去     （读—改—写）
```

而 `PORT_REGS->GROUP[0].PORT_OUTSET = (1UL << 5)` 是**一条写指令**，直接告诉硬件"把第 5 位置 1，其他位不要动"。好处有两个：不会误改别的引脚（哪怕你读到的值已经过时），也不怕被中断打断。这就是 SAMD21 提供这些寄存器的意义。

### 翻转不是第三种电平

`OUTTGL` 并不产生"半高"之类的东西，硬件仍然只有高和低，翻转只是 `0→1`、`1→0`。它适合做 LED 闪烁、方波、状态指示灯。


## 4.8 初始化的顺序：先准备电平，再打开输出

这条规矩是硬件逼出来的。假设 LED 接成"低电平点亮"，而复位后 `OUT` 寄存器是 0：

```c
PORT_REGS->GROUP[0].PORT_DIRSET = LED_MASK;   /* 先打开输出：此刻 OUT=0，LED 立刻全亮 */
PORT_REGS->GROUP[0].PORT_OUTSET = LED_MASK;   /* 再设成灭：LED 闪一下 */
```

现象就是**上电瞬间 LED 闪一下**。正确顺序反过来：

```c
PORT_REGS->GROUP[0].PORT_OUTSET = LED_MASK;   /* 1) 先把输出状态准备成"灭" */
PORT_REGS->GROUP[0].PORT_DIRSET = LED_MASK;   /* 2) 再打开输出驱动，这一瞬间已经是灭的 */
```

一般化：**输出方向打开的那一瞬间，引脚上的电平由 `OUT` 决定**，所以先准备 `OUT`。同样的道理，以后从输出切成输入时，也要先把 `OUT` 设成你想要的上下拉方向，再改 `DIR`。

## 4.9 LED 怎么接：灌电流、限流电阻、驱动强度

LED 有两种接法，**推荐右边**：

```text
拉电流（输出高点亮）              灌电流（输出低点亮，推荐）
GPIO ──[R]──▶|── GND             VDD ──▶|──[R]── GPIO
电流由引脚推出去                  电流被引脚吸进来
```

原因是同一根引脚"能吸"比"能推"多（手册 Table 40-14，VDD = 2.7–3 V 档）：

| `PINCFG.DRVSTR` | 灌电流 IOL（低电平） | 拉电流 IOH（高电平） |
|---|---|---|
| 0（复位默认） | ≤ 2.5 mA | ≤ 2 mA |
| 1（加强驱动） | ≤ 10 mA | ≤ 7 mA |

限流电阻按"LED 压降固定，剩下的电压落在电阻上"来算：

```text
R = (VDD − V_LED) / I        例：3.3V、V_LED=2.0V、5mA
                             R = (3.3 − 2.0) / 0.005 ≈ 260 Ω → 取 220Ω 或 330Ω
```

什么时候打开 `DRVSTR`：普通 LED（3–5 mA）默认就够；驱动光耦、蜂鸣器、长线、多个 LED 并联时打开。但**整颗芯片的总电流还有上限**（手册绝对最大额定值一节），要同时点很多灯，正确做法是用三极管/MOS 或 LED 驱动芯片。

## 4.10 第一个实验：点亮一个 LED

假设 PA05 接 LED，采用"低电平点亮"：

```c
#include "sam.h"

#define LED_PIN  5
#define LED_MASK (1UL << LED_PIN)

static void led_init(void)
{
    /* LED 低电平点亮。先把输出状态设成 1，也就是"灭"。 */
    PORT_REGS->GROUP[0].PORT_OUTSET = LED_MASK;

    /* 再打开输出驱动。顺序不能反，否则上电会闪一下（4.8）。 */
    PORT_REGS->GROUP[0].PORT_DIRSET = LED_MASK;
}

static void led_on(void)
{
    /* OUTCLR：把 PA05 的输出状态设为低 → 灌电流 → LED 亮 */
    PORT_REGS->GROUP[0].PORT_OUTCLR = LED_MASK;
}

static void led_off(void)
{
    /* OUTSET：把 PA05 的输出状态设为高 → LED 灭 */
    PORT_REGS->GROUP[0].PORT_OUTSET = LED_MASK;
}

int main(void)
{
    led_init();

    while (1)
    {
        led_on();
    }
}
```

这段程序虽然短，但已经完整走了一遍"从硬件需求到寄存器"：

```text
硬件需求：LED 低电平点亮  →  GPIO 输出  →  OUT=0  →  DIR=1  →  寄存器 OUTCLR / DIRSET
```

**为什么这里没有配时钟？** 因为 PORT 挂在桥 B 上，APB 时钟复位后就是打开的，而 GPIO 本身不需要通用时钟——所以点灯真的不需要写任何时钟配置。但要知道：**复位后芯片跑得很慢**，OSC8M 八分频，只有 **1 MHz**。这带来两个后果：后面 `delay_ms` 的时间是按 1 MHz 算的；第 5 章把主频提到 48 MHz 后，同一个函数会快 48 倍。

## 4.11 翻转

`OUTTGL` 让掩码里为 1 的引脚取反：

```c
PORT_REGS->GROUP[0].PORT_OUTTGL = LED_MASK;   /* 当前是 0 就变 1，是 1 就变 0 */
```

它不产生第三种电平，只是 `0↔1`。适合做 LED 闪烁、方波输出、状态指示灯——后面流水灯会用到它来"同时熄灭旧的、点亮新的"。


## 4.12 输入：读取一个按键

接法：按键一端接引脚，另一端接 GND，用**内部上拉**提供松开时的默认电平。

```text
            内部上拉（PULLEN=1, OUT=1）
                     │
GPIO ────────────────┴──── 按键 ──── GND

松开 → 上拉 → 读 1        按下 → 接到 GND → 读 0
```

对应 4.5 的四件事，代码是三句（注意最后一句才是"上拉"）：

```c
static void key_init(void)
{
    /* ① 输入缓冲 + ② 上下拉：INEN 让电平能进内部，PULLEN 打开电阻 */
    PORT_REGS->GROUP[0].PORT_PINCFG[8] = PORT_PINCFG_INEN_Msk | PORT_PINCFG_PULLEN_Msk;

    /* ③ 关闭输出驱动 → 输入 */
    PORT_REGS->GROUP[0].PORT_DIRCLR = (1UL << 8);

    /* ④ 上下拉方向由 OUT 决定：OUT=1 → 上拉 */
    PORT_REGS->GROUP[0].PORT_OUTSET = (1UL << 8);
}

static int key_down(void)
{
    /* 按下时被接到 GND，读到 0；松开时上拉，读到 1 */
    return (PORT_REGS->GROUP[0].PORT_IN & (1UL << 8)) == 0;
}
```

`IN` 里是**引脚的实际电平**，所以读之前必须保证 `INEN` 是 1。

## 4.13 多个引脚：bit mask

一个 PORT 组里的寄存器有 32 位，一位对应一个引脚。所以"一组引脚"就是一个 32 位数字：

```text
PA07 PA06 PA05 PA04 PA03 PA02 PA01 PA00
  0    0    0    0    0    0    0    1      = 0x01  → 只有 PA00
  0    0    0    0    0    0    1    0      = 0x02  → 只有 PA01
  0    0    0    0    1    0    0    1      = 0x09  → PA00 和 PA03
```

```c
uint32_t mask = 1UL;                        /* PA00 */
mask = (1UL << 0) | (1UL << 3);             /* PA00 和 PA03 */
mask = 0x000000FFUL;                        /* PA00 ~ PA07 全部 */
```

于是 GPIO 操作永远是两件事的组合：

> **mask 决定"对哪些引脚"，`OUTSET/OUTCLR/OUTTGL` 决定"对它们做什么"。**

## 4.14 流水灯

### 选引脚

例子用 PA00–PA07，只是为了掩码好看（bit0–bit7 连号）。实际接线要避开特殊引脚：

| 引脚 | 特殊用途 |
|---|---|
| PA00 / PA01 | 外部 32.768 kHz 晶振（XIN/XOUT），不用晶振才能当普通 IO |
| PA02 / PA03 | 在 VDDANA（模拟供电域），做数字 IO 可以，但和 ADC/VREF 同域 |
| PA24 / PA25 | USB 的 D− / D+，用 USB 时不能占 |

两条实用规则：**尽量用同一组、连号的引脚**（"掩码移位"就等于"物理上依次移动"）；**换成 PB 的脚时只改两处**——`Group[0]→Group[1]` 和掩码位号。

### 初始化与版本 1：一个亮点依次走过 8 个 LED

```c
#include "sam.h"

#define LED_MASK 0x000000FFUL        /* PA00~PA07，低电平点亮 */

static inline void led_on(uint32_t mask)  { PORT_REGS->GROUP[0].PORT_OUTCLR = mask; }
static inline void led_off(uint32_t mask) { PORT_REGS->GROUP[0].PORT_OUTSET = mask; }

static void leds_init(void)
{
    PORT_REGS->GROUP[0].PORT_OUTSET = LED_MASK;   /* 先准备电平：高 = 灭 */
    PORT_REGS->GROUP[0].PORT_DIRSET = LED_MASK;   /* 再打开输出 */
}

/* 忙等延时：不是精确毫秒，实际时间取决于主频（复位后 1 MHz） */
static void delay_ms(uint32_t ms)
{
    for (uint32_t i = 0; i < ms * 1000U; i++) { __NOP(); }
}

int main(void)
{
    leds_init();

    uint32_t mask = 1UL;                 /* 0000 0001，从 PA00 开始 */
    while (1)
    {
        led_on(mask);                    /* 点亮当前 */
        delay_ms(100);
        led_off(mask);                   /* 熄灭当前 */
        mask <<= 1;                      /* 亮灯位置左移一位 */
        if ((mask & LED_MASK) == 0)      /* 移出 8 个脚就绕回 */
        {
            mask = 1UL;
        }
    }
}
```

流水灯不是凭空出现的算法，它就是 **mask 里的那个 1 在移动**：

```text
0000 0001 → 0000 0010 → 0000 0100 → … → 1000 0000 → 0000 0001
```

### 版本 2：用 OUTTGL 把两次操作合成一次

低电平点亮时，"熄旧的"和"亮新的"恰好都是翻转（旧：0→1，新：1→0），所以可以用一条 `OUTTGL` 写完：

```c
uint32_t mask = 1UL;
led_on(mask);                            /* 先点亮第一个 */

while (1)
{
    delay_ms(100);
    uint32_t next = (mask & 0x80UL) ? 1UL : (mask << 1);
    PORT_REGS->GROUP[0].PORT_OUTTGL = mask | next;   /* 一次翻转两个脚 */
    mask = next;
}
```

### 版本 3：加按键，按住暂停（完整程序）

```c
#include "sam.h"

#define LED_MASK 0x000000FFUL
#define KEY_PIN  8
#define KEY_MASK (1UL << KEY_PIN)

static inline void led_on(uint32_t mask)  { PORT_REGS->GROUP[0].PORT_OUTCLR = mask; }

static void leds_init(void)
{
    PORT_REGS->GROUP[0].PORT_OUTSET = LED_MASK;
    PORT_REGS->GROUP[0].PORT_DIRSET = LED_MASK;
}

static void key_init(void)
{
    PORT_REGS->GROUP[0].PORT_PINCFG[KEY_PIN] = PORT_PINCFG_INEN_Msk | PORT_PINCFG_PULLEN_Msk;
    PORT_REGS->GROUP[0].PORT_DIRCLR  = KEY_MASK;
    PORT_REGS->GROUP[0].PORT_OUTSET  = KEY_MASK;      /* OUT=1 → 上拉 */
}

static int key_down(void)
{
    return (PORT_REGS->GROUP[0].PORT_IN & KEY_MASK) == 0;
}

static void delay_ms(uint32_t ms)
{
    for (uint32_t i = 0; i < ms * 1000U; i++) { __NOP(); }
}

int main(void)
{
    leds_init();
    key_init();

    uint32_t mask = 1UL;
    led_on(mask);

    while (1)
    {
        if (!key_down())                        /* 没按：继续流 */
        {
            delay_ms(100);
            uint32_t next = (mask & 0x80UL) ? 1UL : (mask << 1);
            PORT_REGS->GROUP[0].PORT_OUTTGL = mask | next;
            mask = next;
        }
        /* 按住：什么都不做，灯停在原地 */
    }
}
```

> 这个按键没有做消抖。机械按键按下瞬间会抖动，现象是"按一下偶尔会跳好几步"；消抖可以用延时重读、定时器扫描，或后面学的带滤波的外部中断。


## 4.15 初始化清单

以后遇到 GPIO，按这个顺序想就不会漏：

**输出**：决定初始电平 → 写 `OUT` → 打开 `DIR`。

```c
PORT_REGS->GROUP[0].PORT_OUTCLR = mask;   /* 先准备成低电平 */
PORT_REGS->GROUP[0].PORT_DIRSET = mask;   /* 再打开输出 */
```

**输入**：关输出驱动 → 开输入缓冲 → 可能悬空就配上下拉（含 `OUT` 方向）。

```c
PORT_REGS->GROUP[0].PORT_PINCFG[pin] = PORT_PINCFG_INEN_Msk | PORT_PINCFG_PULLEN_Msk;
PORT_REGS->GROUP[0].PORT_DIRCLR = (1UL << pin);
PORT_REGS->GROUP[0].PORT_OUTSET = (1UL << pin);   /* OUT=1 → 上拉 */
```

**运行中**：只改电平（`OUTSET/OUTCLR/OUTTGL`），不要再动方向和上下拉——那是初始化的事，中途改会产生电气中间态。

## 4.16 一个 GPIO 的完整思维模型

```text
                    GPIO
                      │
       ┌──────────────┼──────────────┐
       │              │              │
      方向           电平           输入
       │              │              │
   DIRSET/CLR     OUTSET/CLR/TGL    INEN
                                     │
                                  PULLEN
                                     │
                                    OUT
                                 （1=上拉 0=下拉）
```

程序真正做的事只有两件：**初始化时配置属性，运行中改电平或读电平**。

## 4.17 常见问题

| 现象 | 原因 | 检查什么 |
|---|---|---|
| 输入一直读不到 | 输入缓冲没开 | `PINCFG.INEN` |
| 松开按键时读数乱跳 | 引脚悬空 | 上拉/下拉 |
| 想要上拉却变成下拉 | `PULLEN` 开了但 `OUT` 没设成 1 | `PULLEN + OUT` |
| 上电 LED 闪一下 | 先打开输出、后设初始电平 | 先 `OUT`，后 `DIRSET` |
| PB 引脚没反应 | Group 写错 | PB 用 `Group[1]` |
| 别的 GPIO 被意外改变 | 用了 `OUT \|= ...` 或者掩码算错 | 改用 `OUTSET/OUTCLR/OUTTGL`，检查 mask |
| `IN` 读不到预期值 | 输入通路没开，或外部电平本身不确定 | `INEN`、上下拉、外部电路 |

## 4.18 寄存器速查

| 寄存器 | 偏移 | 作用 |
|---|---|---|
| `DIR / DIRCLR / DIRSET / DIRTGL` | 0x00 / 0x04 / 0x08 / 0x0C | 方向：0 输入、1 输出 |
| `OUT / OUTCLR / OUTSET / OUTTGL` | 0x10 / 0x14 / 0x18 / 0x1C | 输出电平：当前值 / 低 / 高 / 翻转 |
| `IN` | 0x20 | 只读，引脚实际电平（需 `INEN=1`） |
| `PINCFG[n]` | 0x40 + n | bit1 `INEN`、bit2 `PULLEN`、bit6 `DRVSTR` |

Group 0 = PA（基址 `0x4100_4400`），Group 1 = PB（基址 `0x4100_4480`）。

## 4.19 最重要的几句话

1. GPIO 输出，本质是控制引脚接 VDD 还是接 GND；输入，本质是把引脚电压判成 0 或 1。
2. 输入引脚没有确定外部电平时会悬空，所以需要上拉或下拉。
3. "引脚是输入"和"输入缓冲打开"是两件事：`DIRCLR` 管前者，`INEN` 管后者。
4. SAMD21 的上下拉方向由 `PULLEN + OUT` 决定：`OUT=1` 上拉，`OUT=0` 下拉。
5. 配置输出时先写 `OUT`、再开 `DIR`，可以避免初始化瞬间的错误电平。
6. GPIO 操作 = "对哪些引脚"（mask）＋"做什么动作"（`OUTSET/OUTCLR/OUTTGL`）。
7. 点灯不需要配时钟：PORT 在桥 B，APB 时钟复位后默认已开。

## 4.20 自测

1. 把 PA05 配成"输出、低电平"，要操作哪些寄存器？为什么是这个顺序？
2. 按键接在引脚和 GND 之间，要求松开读 1、按下读 0，为什么必须配内部上拉？
3. 为什么只写 `PULLEN = 1` 不能决定是上拉还是下拉？
4. 为什么输入时要开 `INEN`？不开会怎样？
5. PA00–PA07 接 8 个 LED，为什么 `mask <<= 1` 就能实现流水灯？`mask` 在这里代表什么？
6. 下面两句分别是什么动作？为什么通常按这个顺序？

```c
PORT_REGS->GROUP[0].PORT_OUTSET = mask;
PORT_REGS->GROUP[0].PORT_DIRSET = mask;
```

## 附录：以后查手册怎么找 GPIO

不必一开始通读整个 PORT 章节，按这个顺序查即可：

```text
① 找 PORT 外设  →  ② 找对应 Group  →  ③ 找寄存器名  →  ④ 看 Offset  →  ⑤ 看 bit 定义
```

本章涉及的偏移：`DIR 0x00 / DIRCLR 0x04 / DIRSET 0x08 / DIRTGL 0x0C / OUT 0x10 / OUTCLR 0x14 / OUTSET 0x18 / OUTTGL 0x1C / IN 0x20 / PINCFG[n] 0x40+n`。

## 本章结束

学完这一章，你应该能把 GPIO 代码从"寄存器魔法"翻译成人话。例如

```c
PORT_REGS->GROUP[0].PORT_OUTSET = (1UL << 5);
PORT_REGS->GROUP[0].PORT_DIRSET = (1UL << 5);
```

不是"记住这两句"，而是：**先让 PA05 的输出状态变成高电平，再打开 PA05 的输出驱动**。再例如

```c
PORT_REGS->GROUP[0].PORT_PINCFG[8] = PORT_PINCFG_INEN_Msk | PORT_PINCFG_PULLEN_Msk;
PORT_REGS->GROUP[0].PORT_DIRCLR    = (1UL << 8);
PORT_REGS->GROUP[0].PORT_OUTSET    = (1UL << 8);
```

应该翻译成：**PA08 不再主动输出；打开数字输入通路；打开内部上下拉，并用 OUT=1 选择上拉**。能把寄存器操作翻译成硬件动作，GPIO 才算真正学会。

