
# 第 12 章 中断：NVIC 与 EIC

> **芯片**：ATSAMD21G18A
> **本章目标**：搞清"事件发生时，硬件是怎么打断 CPU 的"，并独立写一个按键中断。
> **学习顺序**：为什么要中断 → 中断在硬件上怎么走 → NVIC 是什么 → EIC 是什么 → 怎么配一个引脚中断 → 寄存器 → 例程 → 排查。
> **前置**：第 4 章（引脚与 PORT）、第 5 章（给外设配时钟）。

## 12.0 为什么要用中断

假设按键接在 PA16，你要"按下时点亮 LED"。最直接的做法是**轮询**：

```c
while (1) {
    if (key_down()) { led_on(); }     /* 每圈都问一次：按下了吗？ */
}
```

它能工作，但有两个毛病：主循环什么都不干也得一直问（浪费 CPU）；如果主循环里还有别的耗时操作，按键按下的瞬间没被看到，**这次按键就丢了**。

**中断**换了个思路：事件发生时，由硬件主动打断 CPU，跳到一段专门的代码去处理，处理完再回到原地继续跑。就像门铃和"每隔 5 秒去门口看一眼"的区别。

## 12.1 一次中断在硬件上怎么走

这是本章的主线，后面所有配置都在为这条链路的某一环服务：

```text
外部事件 → 外设里置起标志 INTFLAG → 外设的中断使能 INTENSET → NVIC → CPU 跳进 ISR
              （事件发生了）              （要不要上报）      （谁先谁后）   （清标志再返回）
```

由此得到三个必须记住的结论：

| 结论 | 为什么 |
|---|---|
| **标志和使能是两回事** | `INTFLAG` 表示"事件发生了"（硬件自动置位），`INTENSET` 表示"要不要上报给 CPU"。只置标志不开使能，什么都不会发生 |
| **标志不清，中断会反复进** | 中断返回前必须清标志；边沿触发时忘了清，会一直卡在同一个中断里 |
| **要开两处** | 外设内部的 `INTENSET` ＋ 内核 NVIC 里的对应线。只开一处，是"中断不进来"最常见的原因 |

## 12.2 NVIC 是什么

NVIC（Nested Vectored Interrupt Controller）在 **Cortex-M0+ 内核里**，不在外设里。它负责：收到哪个外设的请求、和当前正在执行的代码比优先级、决定要不要打断、打断后跳到哪个地址。

| 项目 | SAM D21 上的情况 |
|---|---|
| 中断线数量 | 32 条（手册 11.2.1） |
| 优先级 | **4 级**（0 最高、3 最低）；DFP 头文件里 `__NVIC_PRIO_BITS = 2` |
| 能否嵌套 | 能：高优先级可以打断低优先级 |
| 中断号 | 每个外设固定一条，见附录 14.2（PM=0 … I2S=27） |

三个常用操作（都是 CMSIS 提供的函数，不用自己写寄存器）：

```c
NVIC_EnableIRQ(EIC_IRQn);              /* 允许这条线打断 CPU */
NVIC_SetPriority(EIC_IRQn, 1);         /* 0~3，数字越小越优先 */
NVIC_ClearPendingIRQ(EIC_IRQn);        /* 清掉"已挂起但还没执行"的请求 */
```

**中断服务函数的名字是固定的**：启动文件里已经为每条中断线放好了向量表项，并给了一个"weak 的空实现"：

```c
void EIC_Handler(void) __attribute__((weak, alias("Dummy_Handler")));
```

所以你只要在代码里写一个同名函数（`EIC_Handler`），链接时就会替换掉那个空实现。**如果你拼错了函数名，它会安静地跳到 Dummy_Handler 里——现象就是"中断像是进了但什么都没发生"。** 命名规律就是 `外设名_Handler`：`PM_Handler`、`SERCOM0_Handler`、`TC3_Handler`……

## 12.3 EIC 是什么

EIC（External Interrupt Controller）负责**盯着引脚**：把引脚上的边沿或电平变化，变成一次中断请求。

| 能力 | 说明 |
|---|---|
| 16 条外部中断线 | `EXTINT[0]` ～ `EXTINT[15]`，逐条可单独开关 |
| 1 条 NMI | 不可屏蔽中断，独立一条线，优先级最高 |
| 检测方式 | 上升沿 / 下降沿 / 双边沿 / 高电平 / 低电平（SENSE 六种取值） |
| 滤波 | 每条线可开 FILTEN，抑制按键抖动等毛刺 |
| 低功耗 | 能在睡眠模式下把芯片唤醒（第 6 章展开） |
| 事件输出 | 检测结果还能作为事件送给其它外设（第 11 章） |

### 三个容易踩的概念

**① EXTINT 是"片内信号线"，不是引脚号。** 手册写得很清楚：*"One signal may be available on several pins."* 在这颗芯片上，**PA00 和 PA16 都是 `EXTINT[0]`**，**PA01 和 PA17 都是 `EXTINT[1]`**。同一根线只能服务一个引脚，手册规定：*"If an external interrupt is common on two or more I/O pins, only one will be active (the first one programmed)."* 所以两个按键想用同一根线，只有一个能生效。

**② 引脚必须先"交给 EIC"。** 按手册 7.1 的规则，任何 A–H 复用功能都要先置 `PINCFG.PMUXEN=1`，再用 `PMUX` 选功能；在 Table 7-1 里 **EIC 就在 A 列**。这一步漏了，`DIR`、`INEN`、EIC 寄存器配得再对，中断也不会来。

**③ EIC 是普通外设，也遵守"四个条件"。** 它在桥 A 上：APB 时钟复位后默认已开，但**通用时钟必须自己配**（`GCLK_CLKCTRL_ID_EIC_Val`）；配置寄存器是 Enable-Protected 的，要在 `CTRL.ENABLE=0` 时写；`CTRL` 又是写同步的，改完要等 `STATUS.SYNCBUSY`。这三条和别的外设一模一样。

## 12.4 配一个按键中断的完整流程

需求：PA16 接按键（另一端接 GND），按下时触发中断翻转 LED。

| 步骤 | 做什么 | 为什么 |
|---|---|---|
| 1 | 引脚：`INEN=1` + `PULLEN=1`，`DIRCLR`，`OUTSET`（上拉） | 第 4 章：输入要开缓冲，上拉靠 OUT=1 |
| 2 | 复用：`PINCFG.PMUXEN=1` + `PMUX` 选 **A** | 12.3 的坑②：不交给 EIC，它永远看不到这个脚 |
| 3 | 时钟：`GCLK` 给 EIC 一路通用时钟 | EIC 也要"走时" |
| 4 | EIC：先 `ENABLE=0` → 写 `CONFIG` 的 SENSE 和 FILTEN → 再 `ENABLE=1`，每步等 `SYNCBUSY` | 配置寄存器是 Enable-Protected，`ENABLE` 是写同步 |
| 5 | 开中断：`EIC_REGS->EIC_INTENSET` ＋ `NVIC_EnableIRQ(EIC_IRQn)` | 两处都要开 |
| 6 | ISR：读 `INTFLAG` → **写 1 清除** → 干活 | 不清就反复进 |

把这条链路的每一环对应到 12.1 的图上看，就不会漏。

## 12.5 寄存器逐个讲

EIC 的基址是 `0x40001800`（桥 A）。

| 寄存器 | 偏移 | 它回答什么问题 |
|---|---|---|
| `CTRL` | 0x00 | 整个 EIC 开不开（bit1 ENABLE）、要不要软件复位（bit0 SWRST） |
| `STATUS` | 0x01 | bit7 SYNCBUSY：刚才那次写生效了吗 |
| `CONFIG0` | 0x18 | EXTINT0–7 各自怎么检测 |
| `CONFIG1` | 0x1C | EXTINT8–15 各自怎么检测 |
| `INTENSET` / `INTENCLR` | 0x0C / 0x08 | 哪几条线允许上报 |
| `INTFLAG` | 0x10 | 哪条线触发了（**写 1 清除**） |
| `WAKEUP` | 0x14 | 哪几条线能把芯片从睡眠叫醒 |
| `NMICTRL` / `NMIFLAG` | 0x02 / 0x03 | NMI 的检测方式与标志 |

`CONFIG` 寄存器里，**每条 EXTINT 占 4 位**：低 3 位是 SENSE，第 4 位是 FILTEN。所以 `CONFIG0` 的 bit3:0 管 EXTINT0，bit7:4 管 EXTINT1，依此类推。

SENSE 的六种取值（手册 Table 21-x）：

| SENSE | 含义 | 典型用途 |
|---|---|---|
| 0x0 | NONE | 关闭检测 |
| 0x1 / 0x2 / 0x3 | 上升沿 / 下降沿 / 双边沿 | 按键、编码器 |
| 0x4 / 0x5 | 高电平 / 低电平 | 电平型信号（注意：电平不撤，中断会一直来） |

### 12.5.1 NMI：不可屏蔽中断

EIC 除了 16 条普通线，还有一条 **NMI**（Non-Maskable Interrupt）。它的"不可屏蔽"体现在：**不需要在 NVIC 里使能**，只要 EIC 这边配置好，触发就一定被执行（手册 21.5.5 明确写了 "does not require the interrupt to be configured"）。

| 寄存器 | 偏移 | 说明 |
|---|---|---|
| `NMICTRL` | 0x02 | NMISENSE[2:0]（bit2:0，六种取值同 SENSE）、NMIFILTEN（bit3） |
| `NMIFLAG` | 0x03 | bit0，写 1 清除 |

它适合处理"绝对不能漏"的信号：电源掉电预警、外部看门狗喂狗失败、硬件故障报警。普通按键不要占用它。服务函数名同样是启动文件里定好的：`NonMaskableInt_Handler`。

### 12.5.2 中断进入和返回时，CPU 做了什么

理解这一点，就明白为什么 ISR 必须短、为什么共享变量要加 `volatile`。

```text
事件发生 → 硬件自动把 8 个寄存器压栈（R0-R3、R12、LR、PC、xPSR）
         → 从向量表取出 ISR 地址（取向量）
         → 跳到 ISR；返回时再自动出栈，恢复现场
```

三个由此而来的实际影响：

- **压栈出栈是要花时间的**（十几个时钟周期），中断太频繁会吃掉 CPU；
- **连续两个中断之间不会重复出栈再压栈**（tail-chaining，尾巴链接），所以"中断接中断"比"中断—返回—再中断"快；
- **高优先级中断如果在压栈期间到来，会先执行它**（late arrival），这就是嵌套的硬件基础。

### 12.5.3 EIC 的两个延伸能力

**① 把芯片从睡眠里叫醒**：把 `WAKEUP` 里对应位置 1，这条线就能在低功耗模式下唤醒芯片（第 6 章会用）。EIC 的边沿检测是异步的，睡眠时也能记录。

**② 直接触发别的外设**：`EVCTRL`（偏移 0x04）里每一位对应一条 EXTINT，置 1 后这次检测会作为**事件**送给事件系统，别的外设可以不经 CPU 直接响应（第 11 章展开）。这属于"轮询 → 中断 → 硬件自动联动"的第三层。

### 12.5.4 优先级怎么分配

4 级优先级看着少，按"实时性要求"排序就够用：

| 优先级 | 放什么 | 理由 |
|---|---|---|
| 0（最高） | 电源掉电、通信接收（漏一个字节就错帧） | 漏了无法恢复 |
| 1 | 定时器周期中断、编码器计数 | 要求准时 |
| 2 | 按键、传感器就绪 | 晚几毫秒没影响 |
| 3（最低） | 显示刷新、日志 | 可以等 |

原则：**别把所有中断都设成 0**（那等于没分级），也别在 ISR 里关总中断做长活。



## 12.6 例程：按键中断翻转 LED（完整程序）

PA17 接 LED（低电平点亮），PA16 接按键（另一端接 GND，用内部上拉），按下时翻转 LED。主循环什么都不用做——这正是中断的意义。

```c
#include "sam.h"

#define LED_PIN     17
#define KEY_PIN     16          /* PA16 → 手册 A 列是 EXTINT[0] */
#define KEY_EXTINT  0

#define LED_MASK    (1UL << LED_PIN)
#define KEY_MASK    (1UL << KEY_PIN)

static void led_init(void)
{
    PORT_REGS->GROUP[0].PORT_OUTSET = LED_MASK;   /* 先摆电平：高 = 灭 */
    PORT_REGS->GROUP[0].PORT_DIRSET = LED_MASK;   /* 再打开输出 */
}

static void key_eic_init(void)
{
    /* ① 引脚：输入 + 开输入缓冲 + 开上下拉 */
    PORT_REGS->GROUP[0].PORT_PINCFG[KEY_PIN] = PORT_PINCFG_INEN_Msk | PORT_PINCFG_PULLEN_Msk;
    PORT_REGS->GROUP[0].PORT_DIRCLR = KEY_MASK;
    PORT_REGS->GROUP[0].PORT_OUTSET = KEY_MASK;              /* OUT=1 → 上拉（第 4 章） */

    /* ② 把引脚交给 EIC：先开复用开关，再选功能 A */
    PORT_REGS->GROUP[0].PORT_PINCFG[KEY_PIN] |= PORT_PINCFG_PMUXEN_Msk;
    {
        uint8_t v = PORT_REGS->GROUP[0].PORT_PMUX[KEY_PIN >> 1];
        v = (uint8_t)((v & 0xF0u) | 0x0u);             /* PA16 是偶数脚 → 低 4 位；功能 A = 0 */
        PORT_REGS->GROUP[0].PORT_PMUX[KEY_PIN >> 1] = v;
    }

    /* ③ 给 EIC 一路通用时钟（桥 A 的 APB 时钟复位后已开） */
    GCLK_REGS->GCLK_CLKCTRL = (uint16_t)(GCLK_CLKCTRL_ID_EIC_Val |
                                   GCLK_CLKCTRL_GEN_GCLK0_Val |
                                   GCLK_CLKCTRL_CLKEN_Msk);
    while ((GCLK_REGS->GCLK_STATUS & GCLK_STATUS_SYNCBUSY_Msk)) { }

    /* ④ 配置 EIC：Enable-Protected 的寄存器必须在关闭时写，ENABLE 是写同步的 */
    EIC_REGS->EIC_CTRL &= ~EIC_CTRL_ENABLE_Msk;
    while ((EIC_REGS->EIC_STATUS & EIC_STATUS_SYNCBUSY_Msk)) { }

    EIC_REGS->EIC_CONFIG[0] = EIC_CONFIG_SENSE0_FALL_Val      /* EXTINT0：下降沿（按下） */
                       | EIC_CONFIG_FILTEN0_Msk;         /* 打开滤波，抑制抖动 */

    EIC_REGS->EIC_CTRL |= EIC_CTRL_ENABLE_Msk;
    while ((EIC_REGS->EIC_STATUS & EIC_STATUS_SYNCBUSY_Msk)) { }

    /* ⑤ 开中断：外设侧 + 内核侧，两处都要 */
    EIC_REGS->EIC_INTENSET = (1UL << KEY_EXTINT);
    NVIC_ClearPendingIRQ(EIC_IRQn);                  /* 先清掉可能残留的挂起请求 */
    NVIC_SetPriority(EIC_IRQn, 1);                   /* 0 最高、3 最低 */
    NVIC_EnableIRQ(EIC_IRQn);
}

/* ⑥ 中断服务程序：函数名必须和启动文件里的向量名一致 */
void EIC_Handler(void)
{
    if (EIC_REGS->EIC_INTFLAG & (1UL << KEY_EXTINT))
    {
        EIC_REGS->EIC_INTFLAG = (1UL << KEY_EXTINT);      /* 写 1 清除，不写就会一直进 */
        PORT_REGS->GROUP[0].PORT_OUTTGL = LED_MASK;        /* 翻转 LED */
    }
}

int main(void)
{
    led_init();
    key_eic_init();

    while (1)
    {
        /* 主循环空着也不影响——事件由硬件送进来 */
    }
}
```

> 如果链接时报"找不到 EIC_Handler"，或者下载后按下按键毫无反应，先检查函数名有没有拼错。启动文件里 `EIC_Handler` 是一个 weak 的空实现（`Dummy_Handler`），拼错名字不会报错，只会安静地跳到那个空函数里。

### 12.6.1 例程二：两个中断源同时工作

中断系统真正的价值在于"多个事件各走各的路"。下面让按键中断和 1 毫秒系统节拍（SysTick）同时工作：SysTick 负责计时，按键负责翻转 LED，主循环只做业务。

```c
#include "sam.h"
#include "system_samd21g18a.h"   /* SystemCoreClock 在这里声明；sam.h 默认不带它（`USE_CMSIS_INIT` 未定义时） */

#define LED_PIN 17
#define LED_MASK (1UL << LED_PIN)

static volatile uint32_t ms_tick = 0;      /* 由 SysTick 递增，主循环读 */
static volatile uint8_t  key_event = 0;    /* 由 EIC 置位，主循环消费 */

void SysTick_Handler(void)                 /* 名字同样来自启动文件 */
{
    ms_tick++;                             /* SysTick 的标志由内核硬件清，不用手动清 */
}

void EIC_Handler(void)
{
    EIC_REGS->EIC_INTFLAG = (1UL << 0);         /* PA16 → EXTINT[0] */
    key_event = 1;
}

int main(void)
{
    led_init();                            /* 第 4 章：LED 初始化 */
    key_eic_init();                        /* 12.6 例程一：按键中断初始化 */

    /* SysTick 是内核自带的定时器，不属于 PORT/SERCOM 这类外设，不需要配 GCLK。
       SystemCoreClock 必须等于真实主频：复位后是 1000000，改了主频要同步更新（第 5 章）。 */
    SysTick_Config(SystemCoreClock / 1000);   /* 每 1 ms 中断一次 */

    uint32_t t0 = 0;
    while (1)
    {
        if (key_event) {
            key_event = 0;
            PORT_REGS->GROUP[0].PORT_OUTTGL = LED_MASK;
        }

        if ((uint32_t)(ms_tick - t0) >= 500) {   /* 每 500 ms 做一次别的事 */
            t0 = ms_tick;
        }
    }
}
```

三点说明：

- **SysTick 是 Cortex-M0+ 内核自带的**，所以它不在手册第 12 章的外设表里，也不需要配通用时钟；
- 它的中断号是内核异常（不在 0–27 的外设中断号里），启动文件已经放好了 `SysTick_Handler`；
- 主循环里比较时间要写 `(uint32_t)(ms_tick - t0) >= 500`，**不要写 `ms_tick >= t0 + 500`**——后者在计数器回绕时会出错。


## 12.7 中断服务程序的三条纪律

| 纪律 | 原因 | 做法 |
|---|---|---|
| **短** | ISR 执行期间其它同级或低优先级中断被挡住，主程序也停着 | 只做"清标志 + 记录事件"，重活交给主循环 |
| **清标志** | 边沿触发的中断标志不清，退出后会立刻再进来 | `EIC_REGS->EIC_INTFLAG = 位`（写 1 清除） |
| **共享变量加 `volatile`** | 主循环和 ISR 都可能改它，编译器会做优化 | `static volatile uint8_t flag;` |

一种常用结构（ISR 只置标志，主循环干活）：

```c
static volatile uint8_t key_event = 0;

void EIC_Handler(void)
{
    EIC_REGS->EIC_INTFLAG = (1UL << KEY_EXTINT);
    key_event = 1;                     /* 只记一笔 */
}

int main(void)
{
    led_init(); key_eic_init();
    while (1) {
        if (key_event) {               /* 主循环里做耗时的事 */
            key_event = 0;
            PORT_REGS->GROUP[0].PORT_OUTTGL = LED_MASK;
            delay_ms(200);
        }
    }
}
```

## 12.8 常见问题

| 现象 | 原因 | 检查什么 |
|---|---|---|
| 中断完全不进 | 六环里某一环断了 | ① 引脚复用有没有选功能 A；② `EIC_REGS->EIC_CTRL` 的 ENABLE 位；③ `EIC_INTENSET`；④ `NVIC_EnableIRQ`；⑤ GCLK 有没有给 EIC；⑥ 函数名是否拼对 |
| 进一次就再也不进 | 标志没清 | ISR 里 `EIC_REGS->EIC_INTFLAG = 位` |
| 中断反复进、停不下来 | 电平型检测但电平没撤，或标志没清 | 改成边沿检测，或确认信号已恢复 |
| 按一下进好几次 | 按键机械抖动 | 打开 `FILTEN`，或在软件里延时后重读 |
| 两个按键只有一个能中断 | 两个脚共用同一根 EXTINT 线 | 换到不同的 EXTINT（查手册 Table 7-1 的 A 列） |
| 编译通过、下载后没反应 | 函数名拼错 → 跳进了 weak 空实现 | 对照启动文件里的 `EIC_Handler` |
| 一按就 HardFault | ISR 里访问了没开时钟的外设，或除零 | 检查 ISR 里用到的外设时钟 |
| 高优先级没有打断低优先级 | 优先级没设，或在 ISR 里关了中断 | `NVIC_SetPriority` |

## 12.9 寄存器速查

**EIC**（基址 `0x40001800`）

| 偏移 | 名称 | 关键位 |
|---|---|---|
| 0x00 | CTRL | bit1 ENABLE、bit0 SWRST |
| 0x01 | STATUS | bit7 SYNCBUSY |
| 0x08 / 0x0C | INTENCLR / INTENSET | 每条 EXTINT 一位 |
| 0x10 | INTFLAG | 每条 EXTINT 一位，写 1 清除 |
| 0x14 | WAKEUP | 每条 EXTINT 一位，允许从睡眠唤醒 |
| 0x18 / 0x1C | CONFIG0 / CONFIG1 | 每条 EXTINT 占 4 位：SENSE[2:0] + FILTEN |

**NVIC**（CMSIS 函数，不用直接碰寄存器）

| 函数 | 作用 |
|---|---|
| `NVIC_EnableIRQ(IRQn)` / `NVIC_DisableIRQ(IRQn)` | 允许 / 禁止这条中断线 |
| `NVIC_SetPriority(IRQn, 0..3)` | 设优先级，0 最高 |
| `NVIC_ClearPendingIRQ(IRQn)` | 清掉挂起但还没执行的请求 |

中断号见附录 14.2。

## 12.10 思维模型

```text
        外部信号（按键按下）
                │
        ┌───────┴────────┐
        │  EIC（引脚侧）   │  PMUX 选 A、CONFIG 选边沿、FILTEN 滤波
        └───────┬────────┘
                │ 置 INTFLAG，若 INTENSET 允许
        ┌───────┴────────┐
        │  NVIC（内核侧）  │  比优先级、决定是否打断
        └───────┬────────┘
                │
        ┌───────┴────────┐
        │  EIC_Handler    │  清 INTFLAG → 置标志/干活
        └─────────────────┘
```

配置时照着这张图从下往上问：引脚配了吗？复用选了吗？时钟给了吗？EIC 开了吗？使能开了吗？NVIC 开了吗？标志清了吗？

## 12.11 最重要的几句话

1. 中断是"硬件主动打断 CPU"，解决轮询浪费 CPU 和丢事件的问题。
2. **标志（INTFLAG）和使能（INTENSET）是两件事**；还要加上内核侧的 NVIC，一共两处开关。
3. 中断服务函数的**名字必须和启动文件里的向量名一致**，拼错不会报错，只会跳进空实现。
4. **EXTINT 是片内信号线，不是引脚号**：多个引脚可能共用一根，同一时刻只有一个生效。
5. 引脚必须**先把复用交给 EIC（功能 A）**，否则 EIC 看不到它。
6. ISR 要短、要清标志；跨 ISR 与主循环的变量要加 `volatile`。
7. SAM D21 有 4 级优先级（0 最高），支持嵌套。

## 12.12 自测

1. 为什么"只开 `INTENSET`"中断不会来？还差哪一步？
2. `EIC_REGS->EIC_INTFLAG = 0` 能不能清中断标志？正确写法是什么？
3. PA16 和 PA00 都能做外部中断吗？同时用会怎样？
4. 按键中断里为什么建议只置一个标志、把耗时操作留给主循环？
5. 写出把 EIC 中断优先级设为 1、并使能的 CMSIS 语句。
6. 引脚已经配成"输入 + 上拉"，按键按下却进不了中断，最可能漏了哪一步？

## 附录：查手册怎么找中断相关内容

```text
① 外设章节里的 "Interrupts" 小节  →  这个外设有哪些中断源
② 外设的 INTENCLR / INTENSET / INTFLAG  →  怎么开关与清标志
③ 手册 11.2 与 Table 11-3        →  中断线映射；号数见附录 14.2
④ 手册 21 章（EIC）              →  引脚中断的 SENSE / FILTEN / WAKEUP
⑤ 启动文件 startup_atsamd21g18a.c →  中断服务函数的确切名字
```

