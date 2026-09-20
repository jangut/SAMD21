# 第 3 章 MPLAB 工程与启动流程

> **芯片**：ATSAMD21G18A（Cortex-M0+，256 KB Flash，32 KB SRAM，48 脚 TQFP/QFN，38 个 I/O）
> **工具链**：MPLAB X IDE 6.30 + XC32 5.10 + SAMD21_DFP 3.7.262
> **本章目标**：把"点一下编译按钮"背后的事讲清楚——工程里的三样东西各管什么、DFP 到底给了你哪些文件、复位之后 CPU 走过了哪些代码才到 `main()`，以及为什么复位后主频只有 1 MHz。
> **本章不写**：怎么把时钟配到 48 MHz（第 5 章）。本章只需要你接受一个事实：不配时钟，就只有 1 MHz。
> **学习顺序**：工具链是什么 → 怎么建工程 → DFP 给了什么 → 复位到 main 的完整过程 → 中断向量表 → 链接脚本 → 烧写与调试 → 例程 → 清单 → 思维模型。

## 3.0 本章要回答的问题

| 问题 | 在哪一节 |
|---|---|
| 我装的这三样东西分别干什么？ | 3.1 |
| 新建工程时每一步在选什么，为什么？ | 3.2 |
| DFP 文件夹里那些 .h / .c / .ld 都是给谁用的？ | 3.3 |
| 上电后 CPU 的第一条指令从哪来？ | 3.4 |
| 从复位到 `main()` 中间谁在跑？ | 3.5 |
| 为什么 `SystemInit()` 是空的？ | 3.6 |
| 那张中断向量表长什么样，`weak` 是什么意思？ | 3.7 |
| Flash 和 SRAM 是怎么划分的？ | 3.8 |
| 程序怎么进芯片、调试器怎么连？ | 3.9 |

STM32 那边你多半用过 Keil 或 CubeIDE：新建工程、选器件、点下载，中间那些"模板代码"是 IDE 替你生成的。SAMD21 这边分工不一样——**IDE 不管寄存器，器件相关的一切都放在 DFP（Device Family Pack，器件家族包）里**。所以本章先认清这三个角色，后面每一章的头文件、启动文件、链接脚本都来自 DFP。

## 3.1 工具链现状：三样东西各管什么

| 组件                 | 本机路径                                                          | 负责什么                                    | 缺了它                            |
| ------------------ | ------------------------------------------------------------- | --------------------------------------- | ------------------------------ |
| MPLAB X IDE 6.30   | `C:\Program Files\Microchip\MPLABX\v6.30`                     | 编辑器、工程管理、调试器前端、配置位窗口                    | 只能在命令行编译，没有调试界面                |
| XC32 5.10          | `C:\Program Files\Microchip\xc32\v5.10`                       | 编译器/汇编器/链接器（GCC 13.2.1，含 ARM 后端 pic32c） | 编不出机器码                         |
| SAMD21_DFP 3.7.262 | `...\MPLABX\v6.30\packs\Microchip\SAMD21_DFP\3.7.262\samd21a` | 本器件的头文件、启动文件、链接脚本、配置位定义、烧写脚本            | 找不到 `sam.h`，也不知道 Flash/RAM 有多大 |

```text
main.c ──▶ xc32-gcc（XC32 5.10）
              │  -mdfp= 指向 DFP
              ▼
        DFP 3.7.262：sam.h / startup / .ld
              │
              ▼  app.elf ──▶ 烧进 Flash
```

**XC32 自己不带 SAMD21 的器件文件。** 我查过 `C:\Program Files\Microchip\xc32\v5.10` 下面没有按器件分目录的 `lib\proc\ATSAMD21G18A`，所以 DFP 不是"可选的文档包"，而是编译必需件。两者的分工写在 DFP 的 `xc32\ATSAMD21G18A\ATSAMD21G18A.cfg` 里：

```text
-mcpu=SAMD21G18A
--target=cortex-m0plus
-mfloat-abi=soft
-D__SAMD21G18A__
```

也就是说，**Cortex-M0+ 是 XC32 5.10 的一个目标架构**，`SAMD21G18A` 是这个架构下的一个具体器件。我在命令行用下面这条命令实测过，能直接生成 ELF：

```text
xc32-gcc -mprocessor=ATSAMD21G18A -mdfp=<DFP 路径>\samd21a -O1 main.c -o app.elf
```

MPLAB X 在图形界面里替你拼的就是这条命令。两点提醒：命令行手敲时**路径建议用正斜杠**（`C:/Program Files/...`），反斜杠在有些 shell 下会被吃掉；另外参数顺序和拼写要照抄，`-mprocessor` 少了，驱动就不知道该编哪个器件。所以三者的版本要成套：**IDE 6.30 + XC32 5.10 + DFP 3.7.262**，换任何一个都可能对不上。

## 3.2 新建一个 Standalone Project

流程是 `File → New Project → Microchip Embedded → Standalone Project`，然后按向导一路往下选。菜单文字随版本会略有差别，认准每一步"选的是什么"即可：

| 向导步骤 | 选什么 | 为什么 |
|---|---|---|
| Select Device | `Microchip → SAM D21 → ATSAMD21G18A` | 型号变体决定 Flash/SRAM 大小、引脚数、用哪份链接脚本 |
| Select Tool | 你的调试器（SNAP / Atmel-ICE / PKOB4 / Simulator） | 决定下载和调试走哪条路；只做语法验证可以先选 Simulator |
| Select Compiler | XC32 (v5.10) | 只有 XC32 有 cortex-m0plus 后端 |
| Select Project Name and Folder | 路径尽量纯英文、不带空格 | 工具链处理路径时比较脆弱 |
| Finish | — | 生成 `main.c`、`Makefile`、`nbproject\`（工程配置） |

**为什么选 Standalone Project，而不是 MCC 或 Atmel START 工程？** 因为后两者会先生成一堆初始化代码，你还没搞清启动流程就先面对几百行生成物。这一章的目的是"把裸工程跑通"；等学完第 4 章 GPIO、第 5 章时钟，再决定要不要用代码生成器。

工程建好后只有 `main.c` 一个源文件，内容就是 DFP 模板：

```c
#include "sam.h"

int main(void)
{
    /* Replace with your application code */
    while (1)
    {
    }
}
```

先 Build 一次（应该 0 error），再 Program 一次（确认调试器连得上）。**在没加任何自己的代码之前就下载一次**，是新手最容易省掉、也最省时间的一步：它能证明"器件选对了、工具链齐了、线接对了"，后面再出问题就只剩代码。

## 3.3 DFP 到底给了你什么

打开 `...\SAMD21_DFP\3.7.262\samd21a\`，你只需要记住四行：

```text
samd21a\
├─ include\            samd21g18a.h、sam.h、component\、instance\、pio\
├─ xc32\ATSAMD21G18A\  startup_atsamd21g18a.c、ATSAMD21G18A.ld、.cfg
├─ gcc\                system_samd21g18a.c、gcc\startup、gcc\*_flash.ld
└─ hwtools\ atdf\ edc\ templates\   调试器配置、器件描述、工程模板
```

| 文件 | 是什么 | 你什么时候会看它 |
|---|---|---|
| `include\sam.h` | 总入口：按 `__SAMD21G18A__` 之类的宏挑出具体器件头 | 每个 .c 开头都写 `#include "sam.h"` |
| `include\samd21g18a.h` | 器件头：中断号 `IRQn_Type`、向量表结构 `DeviceVectors`、外设基址宏、`FLASH_SIZE` | 查宏名、查中断号、查外设基址 |
| `include\component\` 下每个外设一份 | 寄存器结构体 + 位定义（外设的"控制开关清单"） | 写寄存器前查真实名字 |
| `include\instance\` 下每个外设一份 | 外设**实例**参数：几个通道、几个引脚、几个组 | 不确定"这器件到底有几个"时 |
| `include\pio\samd21g18a.h` | 引脚宏：`PIN_PA05`、`PORT_PA05`、`MUX_PA05D_SERCOM0_PAD1` | 想知道某个脚能复用成什么 |
| `xc32\ATSAMD21G18A\startup_atsamd21g18a.c` | XC32 的启动文件（复位 → main） | 想知道 `main()` 之前发生了什么 |
| `xc32\ATSAMD21G18A\ATSAMD21G18A.ld` | XC32 的链接脚本（Flash/SRAM 布局） | 改内存布局、查 `_stack`、`_dinit_*` 这些符号 |
| `xc32\ATSAMD21G18A\configuration.data` | 配置位（NVM User Row）的定义，MPLAB X 的 Configuration Bits 窗口读它 | 改 BOOTPROT 这类熔丝位 |
| `gcc\system_samd21g18a.c` | `SystemInit()` / `SystemCoreClock` 的家 | 想把 `SystemCoreClock` 用起来 |
| `gcc\gcc\startup_samd21g18a.c`、`gcc\gcc\samd21g18a_flash.ld` | **另一套**启动文件和链接脚本，给 GCC 命令行工具链用 | 只有用 arm-none-eabi-gcc 时才用 |

**两套启动文件和两套链接脚本，别拿错。** 你用的是 XC32，真正生效的是 `xc32\ATSAMD21G18A\` 那一套；`gcc\` 那一套是给开源 GCC 工具链的，内容相近但不参与你的编译。想对照学习可以看它——GCC 版的启动文件把数据搬运代码明明白白写在 `Reset_Handler` 里，比 XC32 版直观。

**顺手记住这套头文件的命名风格。** 本书统一使用 SAMD21_DFP 3.7.262 的写法：外设基址宏是 `外设名_REGS`，成员名是"外设_寄存器名"，字段宏只有 `_Msk`（掩码）和 `_Val`（枚举值）两种。例如 GPIO 是 `PORT_REGS->GROUP[0].PORT_OUTSET`、`PORT_PINCFG_INEN_Msk`；旧教程里那种 `PORT->Group[0].DIRSET.reg`、`PORT_PINCFG_INEN` 的写法在 3.7.262 上**不存在**，照抄会编译不过。拿不准名字时，打开 `include\component\` 下对应的 .h 文件看一眼。

**为什么工程里看不到启动文件？** 因为编译时由设备 specs 自动加进来。`xc32\ATSAMD21G18A\specs-ATSAMD21G18A` 里有这么几条规则：你没用 `-T` 指定链接脚本，它就自动用 `ATSAMD21G18A.ld`；你没写 `-mno-device-startup-code`，它就把 `startup_atsamd21g18a.c` 一起编译；还会自动加 `-D__SAMD21G18A__`、`-mcpu=cortex-m0plus`、`-mthumb`，并把 DFP 的 `include\` 加进头文件搜索路径。这就是"工程里只有一个 `main.c`，却能生成一张完整向量表"的原因。

## 3.4 复位之后的第一条指令

复位释放的瞬间，CPU 做两件事，这两个地址是硬件写死的（手册 8.3.3 Fetching of Initial Instructions）：

| 地址 | CPU 读走什么 | 装进哪 |
|---|---|---|
| `0x00000000` | 一个 32 位数 | 栈指针 SP |
| `0x00000004` | 一个 32 位数 | 程序计数器 PC（也就是复位处理函数的地址） |

而 `0x00000000` 正是内部 Flash 的起始地址（手册 Table 10-1）。所以**向量表必须待在 Flash 最开头**，否则复位就跑飞。

我把 3.2 节那个空工程真的编译了一次，`objdump` 出来的向量表开头是这样：

| 地址 | 实际内容 | 含义 | 谁提供的 |
|---|---|---|---|
| `0x00000000` | `0x20007FF8` | 初始 SP（栈顶） | 链接脚本的 `_stack` |
| `0x00000004` | `0x000000D1` | `Reset_Handler`（`0xD0`，bit0=1 是 Thumb 标志） | 启动文件 |
| `0x00000008` | `0x000002F3` | NMI → `Dummy_Handler` | 启动文件（weak） |
| `0x0000000C` | `0x000002F3` | HardFault → `Dummy_Handler` | 启动文件（weak） |

表里的函数地址都是奇数（`0xD1`、`0x2F3`），值得解释一下：Cortex-M 只支持 Thumb 指令集，所以**函数地址的最低位固定写 1**，CPU 取地址时会把它当标志位忽略。你在调试器里看到函数地址是奇数，不要以为算错了。

## 3.5 从复位到 main：启动文件做了哪些事

XC32 的 `Reset_Handler` 顺序是这样的：

```text
上电/复位
  → 硬件：SP=[0x0]，PC=[0x4]
  → Reset_Handler：搬 .data、清 .bss
  → SCB->VTOR = 向量表地址
  → __libc_init_array()：C 库初始化
  → main()
```

| 步骤 | 做什么 | 为什么必须有 |
|---|---|---|
| 数据段搬运 | 把存在 Flash 里的初值复制到 SRAM 的 `.data` 区 | `int x = 5;` 的"5"掉电也在 Flash 里，而变量本身住在 SRAM |
| 清零 `.bss` | 把没有初值的全局变量区全部写 0 | C 语言规定全局变量初值为 0，可 SRAM 上电内容是随机的 |
| 设置 `SCB->VTOR` | 告诉内核"向量表在哪儿" | 向量表可以被重定位（比如做 Bootloader 时挪到别处） |
| `__libc_init_array()` | 跑 C 库初始化、调用全局构造函数 | C++ 全局对象、`__attribute__((constructor))` 靠它 |
| 调 `main(0, NULL)` | 把控制权交给你的代码 | 之后正常就不返回了 |

数据段搬运是这里最值得画一张图的动作：

```text
Flash（掉电不丢）                 SRAM（掉电就丢）
0x190 起的 .dinit 表  ──搬──▶   .data：有初值的变量
                                  .bss ：无初值的变量（清 0）
```

两套启动文件的写法不同，结果一样：XC32 版只调用一句 `__pic32c_data_initialization()`，真正的搬运代码在 `libpic32c` 库里，按链接器生成的 `.dinit` 表干活（表里的动作有 COPY、CLEAR、COPY_VAL_EMB、COMPRESSED 几种）；GCC 版则在 `Reset_Handler` 里用 `_etext → _srelocate`、`_szero → _ezero` 这几对符号写了一段显式循环。**所以"启动文件"不只是开头那几十行汇编，搬运数据也是它的一部分。**

启动文件还留了三个 hook（钩子）函数，都是 `weak` 的，你在任何 .c 里定义同名函数就能插进启动流程：

| 你定义的函数 | 什么时候被调用 |
|---|---|
| `_on_reset()` | 在数据段搬运之前（此时全局变量还没有初值，别读它们） |
| `_on_bootstrap()` | 在 C 库初始化之后、`main()` 之前 |
| `_on_exit()` | `main()` 返回之后（正常程序不该走到这里） |

**一个必须知道的细节**：DFP 3.7.262 的 XC32 启动文件里**没有调用 `SystemInit()`**。我把整个 DFP 搜了一遍，`SystemInit` 只有定义（`gcc\system_samd21g18a.c`）和声明（`include\system_samd21g18a.h`），没有任何调用点。这意味着什么，下一节说。

## 3.6 SystemInit 为什么是空的

`gcc\system_samd21g18a.c` 的正文其实就这几行：

```c
#define __SYSTEM_CLOCK    (1000000)
uint32_t SystemCoreClock = __SYSTEM_CLOCK;   /* 当前主频，复位后就是 1 MHz */

void SystemInit(void)
{
    SystemCoreClock = __SYSTEM_CLOCK;        /* 只对齐这个变量，不配置任何硬件 */
}
```

为什么它可以这么空？因为**复位后的时钟是硬件默认值**。手册 14.8 Clocks after Reset 写得很清楚：OSC8M（内部 8 MHz 振荡器）被 8 分频，GCLK0 用它产生 GCLK_MAIN，CPU 和总线都不再分频。8 MHz ÷ 8 = **1 MHz**。

| 事实 | 含义 |
|---|---|
| 复位默认 OSC8M ÷ 8 | CPU 主频 1 MHz |
| `SystemCoreClock = 1000000` | 这个变量描述"当前主频是多少"，默认值恰好等于事实 |
| `SystemInit()` 什么都不配 | 硬件已经是这个状态，它没有活可干 |
| XC32 启动文件不调用它 | 调用不调用，结果都是 1 MHz |

于是有三个后果，一个比一个重要：

**第一，延时/超时/波特率全都按 1 MHz 算。** 你在别处看到的 48 MHz 是这颗芯片的上限，不是默认值。第 4 章那个忙等延时函数如果按 48 MHz 估算循环次数，实际会慢 48 倍；反过来，按 1 MHz 写的延时，等第 5 章把主频提到 48 MHz 之后会快 48 倍。

**第二，`SystemCoreClock` 不会自动出现在你的工程里。** 默认工程只编译 `main.c` 加启动文件，`system_samd21g18a.c` 需要你手动 `Add Existing Item` 加进来（就在 DFP 的 `gcc\` 下）；不加，链接时会报 `undefined reference to 'SystemCoreClock'`。

**第三，第 5 章配完时钟后必须自己更新 `SystemCoreClock`。** 它是给库函数和你的代码看的"当前主频"，硬件不会替你改。改了时钟却忘了改它，就会出现"代码看起来对、时间全不对"的问题。

## 3.7 中断向量表与 weak 符号

向量表就是"**中断号 → 处理函数地址**"的数组，紧跟在复位向量后面。它的成员顺序定义在 `include\samd21g18a.h` 的 `DeviceVectors` 结构体里：前 16 项是 Cortex-M0+ 内核固定的异常，之后是外设，顺序和手册 Table 11-3 的 Interrupt Line Mapping 一致。

| 表内位置 | 处理函数名 | 谁的中断 |
|---|---|---|
| 0 | `pvStack`（不是函数，是初始 SP） | — |
| 1 | `Reset_Handler` | 复位 |
| 2 / 3 | `NonMaskableInt_Handler` / `HardFault_Handler` | 内核异常 |
| 4–10 / 11 | 保留 / `SVCall_Handler` | 内核异常 |
| 12–13 / 14 / 15 | 保留 / `PendSV_Handler` / `SysTick_Handler` | 内核异常 |
| 16 + n 起 | `pfn<外设>_Handler` | 外设中断号 n |

外设那一段（n = 中断号）是这样排的：

| n | 外设 | n | 外设 | n | 外设 |
|---|---|---|---|---|---|
| 0 | PM | 9–14 | SERCOM0–5 | 21、22 | **保留** |
| 1 | SYSCTRL | 15–17 | TCC0–2 | 23 | ADC |
| 2 | WDT | 18–20 | TC3–5 | 24 | AC |
| 3 | RTC | | | 25 | DAC |
| 4 | EIC | | | 26 | PTC |
| 5 | NVMCTRL | | | 27 | I2S |
| 6 | DMAC | | | | |
| 7 | USB | | | | |
| 8 | EVSYS | | | | |

注意 21、22 是空的。手册 Table 11-3 在这个位置写的是 TC6/TC7，但那是家族里别的封装型号才有的定时器，ATSAMD21G18A 上不存在，所以 DFP 里这两个位置就叫 `pvReserved21` / `pvReserved22`。**手册列的是整个家族，DFP 头文件列的才是你这颗芯片。**

**`weak`（弱符号）是什么意思？** 启动文件里每个处理函数都是这样声明的：

```c
void HardFault_Handler(void) __attribute__ ((weak, alias("Dummy_Handler")));
```

翻译成人话：**"每个中断我都准备了一个默认实现，名字就叫这个；如果没人反对，它指向 `Dummy_Handler`（一个 `while(1)` 死循环）。"** 你在自己的 .c 里写一个同名函数，链接器会优先用你的强符号：

```text
外设置位 INTFLAG → NVIC → 查向量表[16 + n]
                              │
                     weak 默认：Dummy_Handler
                     你写了同名函数 → 用你的
```

所以"顶替一个中断处理函数"不需要改启动文件、不需要注册回调，**写一个同名函数就够了**。本章的例程就用这招给 HardFault 加一个停机点；中断怎么使能、优先级怎么设，属于第 12 章。

## 3.8 链接脚本：Flash 和 SRAM 怎么分

链接脚本（`.ld`）回答三个问题：芯片有哪些内存、每段代码/数据放哪儿、各个符号（`_etext`、`_stack`……）是多少。XC32 用的 `ATSAMD21G18A.ld` 里，内存区域是这样声明的：

| 区域名 | 起始 | 长度 | 装什么 |
|---|---|---|---|
| `rom` | `0x00000000` | `0x40000`（256 KB） | 向量表、代码、常量、`.data` 的初值副本 |
| `ram` | `0x20000000` | `0x8000`（32 KB） | `.data`、`.bss`、堆、栈 |
| `config_00804000` / `config_00804004` | `0x00804000` / `0x00804004` | 各 4 字节 | NVM User Row（配置位） |

还是那个空工程，链接后几个关键符号的实际值（你的程序变大后会变，**以工程 `dist\...\production\*.map` 为准**）：

| 符号 | 实测值 | 含义 |
|---|---|---|
| `__svectors` / `exception_table` | `0x00000000` | 向量表位置，也就是 VTOR 要指向的地址 |
| `_dinit_addr` / `_dinit_size` | `0x190` / `0x60` | 数据初始化表的位置和长度，库函数按它搬 `.data` |
| `_sbss` / `_ebss` | `0x20000000` | `.bss` 的起止（本例没有全局变量，所以是空的） |
| `_stack` | `0x20007FF8` | 初始栈顶，和向量表第 0 个字一模一样 |
| `__ram_end` | `0x20008000` | SRAM 末尾再往后一格 |
| `__rom_end` | `0x00040000` | Flash 末尾再往后一格 |

表里故意没有 `_etext`：XC32 链接脚本里的 `_etext` 只标在开头几段固定段的结尾（那个空工程实测是 `0x000000D0`），真正的函数由链接器分配在它后面（同一个工程里 `main` 在 `0x000002C6`）。**想知道某个函数到底放在哪，看 map 文件，别拿 `_etext` 当代码结束地址。**

从表里能读出两条硬件常识。第一条：**栈从高地址往低地址长**，栈顶在 SRAM 最末尾（`0x20007FF8`），所以栈溢出会往低地址踩，先踩堆、再踩 `.bss`/`.data`。现象可能是"某个全局变量莫名其妙变了"，也可能是 HardFault——这也是为什么调试时看到"变量自己变了"要先怀疑栈。第二条：**`.data` 有两个地址**——它占用 SRAM 的地址（运行时用），初值却存在 Flash 里（下载时写），启动文件负责把后者搬到前者。

顺带对比 `gcc\gcc\samd21g18a_flash.ld`：那份脚本把栈和堆写成了显式大小（`STACK_SIZE = 0x400`、`HEAP_SIZE = 0x200`），还定义了 `_sstack` / `_estack` 符号；XC32 这份脚本没写死栈大小，栈顶由链接器放在 RAM 末尾（本例是 `_stack`，最小栈大小符号是 `_min_stack_size`）。**要改内存布局**（比如做 Bootloader，把应用挪到 `0x2000` 之后），GCC 版脚本的注释给了现成做法：给链接器传 `--defsym=ROM_ORIGIN=...`、`--defsym=ROM_LENGTH=...`；XC32 工程则是在工程属性里给 `xc32-ld` 加同样的参数。

## 3.9 烧写与调试：DSU、SWD 两线、Memory 窗口

**先接线。** SAMD21 的编程/调试接口只有一种：SWD（Serial Wire Debug，串行线调试），两根本线加电源和地（手册 Table 7-4）：

| 信号 | 芯片引脚 | 说明 |
|---|---|---|
| SWCLK | PA30 | 调试时钟，由调试器输出 |
| SWDIO | PA31 | 双向数据线 |
| GND / VTref | — | 地，以及目标电压参考（调试器靠它判断电平） |
| RESET | RESET 脚 | 可选，但接上更省事：调试器可以先复位再接管 |

两个电气细节值得记住：手册 45.7 节特别注明"SWCLK 上的上拉对可靠连接很关键"，Table 45-9 给的推荐值是 SWDCLK、SWDIO 各接 **15 kΩ 上拉**；另外 SWCLK 默认就归调试系统（DSU）使用，而 SWDIO 平时是普通 IO，**调试器靠冷插拔/热插拔检测时自动把它切到 SWD 功能**（手册 7.2.2）。反过来说，如果你把 PA30 复用成别的东西，热插拔检测就会失效，得断电或复位才能恢复。

**DSU 是什么？** 全称 Device Service Unit（器件服务单元，手册第 13 章）。它管四件事：

| 能力 | 用处 |
|---|---|
| 调试探针检测（冷插拔/热插拔） | 调试器一插上就能被发现，不用先复位 |
| CPU 复位延长（CPU Reset Extension） | 复位放开后先把 CPU 按住，等调试器接上再放它跑，避免"程序已经跑飞了才连上" |
| Chip-Erase / CRC32 | 整片擦除（芯片被锁死时的救命手段）、内存 CRC 校验 |
| CoreSight 器件识别 | 调试器读它来确认"我连的是什么芯片" |

**常用工具**：Atmel-ICE、SNAP、PKOB4、SAM D21 Xplained Pro 板载 EDBG，走 SWD 都一样（手册 45.7 节）。工具是在 3.2 节向导的第一步选好的，之后随时可以在工程属性里改。

**烧写与调试的三种用法**（工具栏按钮）：

| 动作 | 做什么 | 什么时候用 |
|---|---|---|
| Build | 只编译 | 改代码后先看有没有语法错 |
| Program | 烧进 Flash 并复位运行 | 平时下载 |
| Debug | 烧写后停在 `main()`，进入调试界面 | 要单步、看变量、看寄存器 |

**调试时最有用的四个窗口**（都从 `Window` 菜单打开）：

| 窗口 | 看什么 | 举例 |
|---|---|---|
| Memory | 按**地址**看内存，可选字节/半字/字 | 输入 `0x00000000` 看向量表；输入 `0x40000C00` 看 GCLK 寄存器、`0x41004400` 看 PORT |
| Watches | 按**变量名**看值，实时刷新 | 看 `SystemCoreClock` 是不是 1000000 |
| Variables | 当前作用域的局部变量 | 单步时看循环变量 |
| Disassembly | C 代码对应的汇编 | 确认编出来的是 Thumb 指令、`main` 从哪开始 |

"用 Memory 窗口直接看寄存器"这一招值得单独强调：SAMD21 的外设不是"函数"，而是一段物理地址（第 2 章）。想知道某个外设现在什么状态，不必写 printf——**在 Memory 窗口里输入它的基址加偏移，直接看数值**。地址从哪来？`include\samd21g18a.h` 里每个外设都有 `XXX_REGS` 基址宏，`include\component\xxx.h` 里每个寄存器都有偏移。

## 3.10 完整例程：证明它真的跑在 1 MHz 上

**硬件需求**：一块 ATSAMD21G18A 板子 + 任意 SWD 调试器。不需要 LED、不需要按键，本例全部用调试器观察。

要证明三件事：复位后主频是 1 MHz；启动文件确实处理过全局变量；`weak` 处理器可以被顶替。

### 版本 1：不依赖任何 DFP 源文件

```c
#include "sam.h"

/* 复位后主频 1 MHz（OSC8M 8 MHz ÷ 8）。这里写死，因为它就是硬件默认值 */
#define F_CPU_HZ   1000000UL

volatile uint32_t g_ticks;      /* 无初值 → 启动文件把它所在的 .bss 清 0 */
volatile uint32_t g_seed = 7;   /* 有初值 → 初值存在 Flash，启动时搬到 SRAM */

/* 顶替启动文件里的 weak 版本：出问题时停在这里，而不是死在 Dummy_Handler 里 */
void HardFault_Handler(void)
{
    while (1) { __NOP(); }      /* 调试时在这里打断点，看调用栈就知道谁踩了内存 */
}

static void delay_ms(uint32_t ms)
{
    /* 忙等：循环次数按 1 MHz 估算，只做"时间量级"的验证，不是精确毫秒 */
    for (uint32_t i = 0; i < ms * (F_CPU_HZ / 1000UL); i++) { __NOP(); }
}

int main(void)
{
    g_ticks = 0;                /* 覆盖 .bss 的清零结果，便于观察 */

    while (1)
    {
        g_ticks++;              /* 在 Watches 窗口里看它是否每秒涨约 2 次 */
        delay_ms(500);
    }
}
```

观察步骤：

| 步骤 | 操作 | 应该看到 |
|---|---|---|
| 1 | Debug 进入，程序停在 `main()` | — |
| 2 | Memory 窗口输入 `0x00000000` | 第一个字是栈顶地址，第二个字是 `Reset_Handler`（奇数） |
| 3 | Watches 加 `g_seed` | 值就是 7 —— 说明 `.data` 搬运成功 |
| 4 | Watches 加 `g_ticks`，全速运行几秒 | 数值每秒涨一两次 —— 说明 `delay_ms(500)` 是半秒量级，不是几毫秒 |
| 5 | 把 `F_CPU_HZ` 改成 `48000000UL` 再跑 | 数值每秒涨近百次（约 48 倍）—— 这就是"时钟没配却按 48 MHz 算"的后果 |

### 版本 2：把 SystemCoreClock 用起来

把 DFP 的 `gcc\system_samd21g18a.c` 通过 `Add Existing Item` 加入工程，然后：

```c
#include "sam.h"

extern uint32_t SystemCoreClock;    /* 由 system_samd21g18a.c 定义，复位后是 1000000 */

volatile uint32_t g_ticks;

static void delay_ms(uint32_t ms)
{
    /* 用 SystemCoreClock 而不是写死的数：第 5 章改了主频，这里自动跟着变 */
    for (uint32_t i = 0; i < ms * (SystemCoreClock / 1000UL); i++) { __NOP(); }
}

int main(void)
{
    while (1)
    {
        g_ticks++;
        delay_ms(500);
    }
}
```

在 Watches 里看 `SystemCoreClock`，值是 `1000000`——**这就是"复位后 1 MHz"最直接的证据**。等你第 5 章把主频配到 48 MHz，要在配完时钟后把这个变量更新成 `48000000UL`，否则上面这个延时函数仍然按 1 MHz 数循环。

## 3.11 新建工程后的初始化清单

以后每建一个新工程，按这个顺序过一遍就不会踩坑：

| # | 检查项 | 怎么检查 |
|---|---|---|
| 1 | 器件是 ATSAMD21G18A | 工程属性 → Device；选错变体会用错链接脚本 |
| 2 | 工具链是 XC32 5.10 | 工程属性 → Compiler Toolchain |
| 3 | DFP 已安装且版本成套 | `Tools → Packs`，看 SAMD21_DFP 3.7.262 |
| 4 | 先空工程编译 + 下载一次 | 通了说明硬件链路没问题 |
| 5 | 要用 `SystemCoreClock` 就加 `gcc\system_samd21g18a.c` | 不加会 `undefined reference` |
| 6 | 需要更快主频 → 第 5 章配时钟，并更新 `SystemCoreClock` | 本章只负责让你知道默认是 1 MHz |
| 7 | 要用的中断 → 写同名处理函数顶替 weak 版本 | 中断使能与优先级见第 12 章 |
| 8 | 要看寄存器 → Memory 窗口 + DFP 的基址宏和偏移 | 见 3.9 |

## 3.12 一个工程的思维模型

```text
                      app.elf
                         │
        ┌────────────────┼─────────────────┐
      Flash            SRAM           NVM User Row
   （向量表+代码）  （.data/.bss/栈）   （配置位）
   复位 → Reset_Handler → main()，主频 1 MHz，等你配时钟
```

图的上半是"东西放在哪"：代码和初值在 Flash，变量和栈在 SRAM，配置位在 NVM User Row；最下面一行是"上电后谁在跑"：复位向量 → 启动文件 → `main()`。把这两件事串起来的是 DFP 提供的三份文件——头文件（名字）、启动文件（搬运）、链接脚本（布局）。**工程出问题时，先问自己"是文件不对、是搬运不对，还是布局不对"。**

## 3.13 常见问题

| 现象 | 原因 | 检查什么 |
|---|---|---|
| 向导里找不到 ATSAMD21G18A | DFP 没装或装的是别的系列 | `Tools → Packs` 里找 SAMD21_DFP |
| 编译器下拉框是空的 | XC32 没装，或 IDE 没扫描到 | `Tools → Options → Embedded → Build Tools` 里添加/扫描 |
| `fatal error: sam.h: No such file or directory` | 器件选错，或 DFP 版本与工具链不匹配 | 工程属性的 Device、`-mdfp` 指向的 DFP 路径 |
| 编译通过，下载失败 | 接线、供电、SWCLK 被复用、芯片被锁 | 先确认 SWCLK/SWDIO/GND/VTref，再试 Chip-Erase |
| 选错器件变体（换成 G18AU 之类） | 内存大小/引脚不同，链接脚本跟着换 | 工程属性里的 Device，改回 ATSAMD21G18A |
| 延时比预期快/慢约 48 倍 | 时钟没配就按 48 MHz 估算，或改了时钟没更新 `SystemCoreClock` | 见 3.6 与第 5 章 |
| 复位后全局变量不是初值 | 自己改过启动文件/链接脚本，或工程里有两份启动文件 | 确认只有 DFP 提供的一份启动文件 |
| 旧教程的代码编译不过（`PORT->Group[0].DIRSET.reg` 之类） | 不同 DFP 版本的寄存器命名不同 | 按 3.3 的写法，打开 `include\component\port.h` 看这版真实名字 |
| `undefined reference to 'SystemCoreClock'` | `system_samd21g18a.c` 没加进工程 | 见 3.6、3.10 版本 2 |

## 3.14 速查

| 想找什么 | 在哪 |
|---|---|
| 复位向量 / 初始 SP | Flash `0x00000000`（SP）、`0x00000004`（PC） |
| 向量表结构定义 | `include\samd21g18a.h` 的 `DeviceVectors` |
| 中断号 | `include\samd21g18a.h` 的 `IRQn_Type`；手册 Table 11-3 |
| 启动文件（XC32 用） | `xc32\ATSAMD21G18A\startup_atsamd21g18a.c` |
| 链接脚本（XC32 用） | `xc32\ATSAMD21G18A\ATSAMD21G18A.ld` |
| `SystemInit` / `SystemCoreClock` | `gcc\system_samd21g18a.c`（需手动加入工程） |
| 外设基址宏 | `include\samd21g18a.h` 的 `XXX_REGS` |
| 寄存器偏移与位定义 | `include\component\<外设>.h` |
| 引脚与复用宏 | `include\pio\samd21g18a.h` |
| SWD 引脚 | SWCLK = PA30，SWDIO = PA31（手册 Table 7-4） |
| 复位后主频 | 1 MHz（OSC8M 8 MHz ÷ 8，手册 14.8） |

## 3.15 最重要的几句话

1. MPLAB X 管界面、XC32 管编译、DFP 管器件；**SAMD21 的器件文件全在 DFP 里，缺了它编不了**。
2. 复位时硬件从 `0x00000000` 取栈顶、从 `0x00000004` 取复位函数地址，所以**向量表必须待在 Flash 最开头**。
3. 启动文件做的事：搬 `.data`、清 `.bss`、设 `VTOR`、初始化 C 库、调 `main()`——`main()` 不是起点。
4. 复位后主频是 **1 MHz**（OSC8M ÷ 8）；`SystemInit()` 是空的，因为它无需改变现状，只把 `SystemCoreClock` 对齐到事实。
5. `SystemCoreClock` 不自动进工程；改了时钟之后必须自己更新它。
6. 中断处理函数都是 `weak` 的，默认指向 `Dummy_Handler`；**写一个同名函数就顶替掉了**。
7. `.data` 存在 Flash、跑在 SRAM，栈从 SRAM 末尾往下长，栈溢出会踩变量——这些都由链接脚本决定。
8. 调试靠 SWD 两线（PA30/PA31）加 DSU；Memory 窗口输入地址就能直接看寄存器。

## 3.16 自测

1. 新建工程时，"选器件"和"选编译器"分别决定了什么？为什么这两步不能随便选？
2. 复位释放后 CPU 从哪两个地址取什么？如果把向量表挪到 `0x2000` 开头，程序还能正常复位运行吗？为什么？
3. `int x = 5;` 这个变量在下载进芯片之后、`main()` 执行之前，分别存在哪里？谁负责让它变成 5？
4. 为什么 DFP 里的 `SystemInit()` 几乎是空的？如果你的程序里延时全部偏快 48 倍，最可能的原因是什么？
5. 你在工程里写了 `void TC3_Handler(void) { ... }`，启动文件里也有一份 `TC3_Handler`，链接时用谁的？为什么？
6. 空工程链接后 `_stack = 0x20007FF8`、`__ram_end = 0x20008000`。这两个数说明栈在哪里、朝哪个方向长？栈用超了会发生什么？

## 附录：以后查手册和查文件怎么找

| 想查什么 | 去哪查 |
|---|---|
| 复位后时钟是多少 | 手册 14.8 Clocks after Reset |
| 复位从哪取指令 | 手册 8.3.3 Fetching of Initial Instructions |
| 中断号对应谁 | 手册 Table 11-3，再用 DFP 的 `IRQn_Type` 核对本器件 |
| 宏名/寄存器叫什么 | DFP 的 `include\component\`、`include\instance\`、`include\pio\` |
| Flash/SRAM 多大 | 手册 Table 10-1，再看 DFP 的 `ATSAMD21G18A.ld` |
| 调试口怎么接 | 手册 Table 7-4（引脚）、45.7 节（连接与上拉） |
| 工程实际占多少内存 | 工程 `dist\...\production\*.map` |

一句话原则：**手册告诉你"硬件是什么"，DFP 告诉你"这颗芯片上叫什么名字"，链接后的 map 文件告诉你"你的程序实际占了多少"。** 三者对不上时，先信 DFP 和 map。
