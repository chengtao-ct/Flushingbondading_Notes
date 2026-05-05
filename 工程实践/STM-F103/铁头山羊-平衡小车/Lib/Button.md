#### Button层的封装

这种分层架构是工业界的标准做法：驱动层只管硬件和状态检测，不关心业务；用户层只管业务逻辑，不关心硬件细节。中间靠回调函数桥接。你以后写任何外设驱动都可以套这个模板。



好，我换个角度，不讲逐行逻辑了，而是站在架构设计的高度，带你理解这个库"为什么这样设计"。
---
一、架构分层：三层分离
┌─────────────────────────────────────────────┐
│  第三层：业务层 (app_button.c)               │
│  定义"按下后做什么"                           │
│  单击→切换电机  双击→开关灯  长按→校准        │
├─────────────────────────────────────────────┤
│  第二层：驱动层 (button.c / button.h)        │
│  定义"怎么检测按键事件"                       │
│  消抖、长按、连击、回调分发                    │
├─────────────────────────────────────────────┤
│  第一层：硬件层 (GPIO + Systick)             │
│  读引脚电平、提供时间基准                      │
└─────────────────────────────────────────────┘
设计原则：每一层只做自己该做的事，不知道上层的存在。
- 驱动层（button.c）不知道"电机""灯""校准"是什么
- 业务层（app_button.c）不知道消抖要等10ms、长按阈值是1000ms
- 两者通过回调函数这个"插座"连接
---
二、头文件的接口设计哲学
看 button.h 对外暴露了什么：
// 生命周期管理
void My_Button_Init(Button_TypeDef *Button, Button_InitTypeDef *Init);
void My_Button_Proc(Button_TypeDef *Button);
// 状态查询
uint8_t MyButton_GetState(Button_TypeDef *Button);
// 回调注册
void My_Button_SetPressCb(...);
void My_Button_SetReleaseCb(...);
void My_Button_SetClickCb(...);
void My_Button_SetLongPressCb(...);
// 参数配置
void My_Button_ClickIntervalConfig(...);
void My_Button_LongPressConfig(...);
老兵经验：一个好的C语言模块接口，通常遵循这个模式：
Init()    → 初始化，只能调一次
Proc()    → 周期调用，放在主循环里
SetXxx()  → 可选配置，在Init之后调
GetXxx()  → 状态查询
你的项目里几乎所有模块都遵循这个模式：
App_Motor_Init() / App_Motor_Proc()
App_Bat_Init()   / App_Bat_Proc()
App_Control_Init()/ App_Control_Proc()
这不是巧合，是嵌入式裸机架构的标准范式。
---
三、回调函数机制 — 为什么要这样设计
对比两种写法
写法A（坏）：硬编码业务逻辑
// button.c 里直接写死业务
if(Clicked) {
    App_Motor_Cmd(TOGGLE);  // 驱动层直接调业务层
}
写法B（好）：回调函数（本库的做法）
// button.c 里
if(Clicked) {
    if(button_clicked_cb)  // 不关心谁注册的，调就是了
        button_clicked_cb(ClickCnt);
}
// app_button.c 里
My_Button_SetClickCb(&UserButton, UserButtonClickCb);  // 业务层注册
区别：
 	写法A（硬编码）
复用性	只能用于这一个项目
依赖关系	驱动层依赖业务层（反向依赖）
编译耦合	改业务要重编驱动
回调的本质是控制反转（IoC）：驱动层不调业务层，而是业务层把函数"注入"驱动层，由驱动层在合适的时机回调。这是嵌入式里最常用的解耦手段。
---
四、对象化封装 — 一个结构体就是一个按钮
typedef struct {
    GPIO_TypeDef *GPIOx;
    uint16_t GPIO_Pin;
    // ... 状态变量
} Button_TypeDef;
所有库函数的第一个参数都是 Button_TypeDef *Button：
void My_Button_Init(Button_TypeDef *Button, ...);
void My_Button_Proc(Button_TypeDef *Button);
void My_Button_SetClickCb(Button_TypeDef *Button, ...);
这等价于C++的 this 指针：
// C语言写法              // C++等价写法
My_Button_Proc(&btn1);   // btn1.Proc();
My_Button_Proc(&btn2);   // btn2.Proc();
这样做的好处：你可以创建任意多个按钮对象，互不干扰：
Button_TypeDef btn1, btn2, btn3;  // 三个按钮
My_Button_Init(&btn1, ...);       // 各自初始化
My_Button_Init(&btn2, ...);
My_Button_Init(&btn3, ...);
My_Button_SetClickCb(&btn1, cb1); // 各自注册不同回调
My_Button_SetClickCb(&btn2, cb2);
My_Button_SetClickCb(&btn3, cb3);
while(1) {
    My_Button_Proc(&btn1);  // 各自独立运行
    My_Button_Proc(&btn2);
    My_Button_Proc(&btn3);
}
老兵经验：这就是C语言实现"面向对象"的标准手法——结构体当对象，函数第一个参数当this。Linux内核、FreeRTOS源码里到处都是这种写法。掌握了这个模式，你读任何工业级C代码都不会陌生。
---
五、.h 文件的封装边界
button.h 只暴露了用户需要看到的：
// 用户需要创建对象 → 暴露 Button_TypeDef
typedef struct {
    GPIO_TypeDef *GPIOx;
    uint16_t GPIO_Pin;
    void (*button_pressed_cb)(void);
    // ... 全部成员
} Button_TypeDef;
老兵经验：严格来说，这个库的封装做得不够彻底。工业级做法是：
// button.h — 对外只暴露一个不透明指针
typedef struct Button_TypeDef Button_TypeDef;  // 前向声明，不暴露内部
// button.c — 内部才知道完整定义
struct Button_TypeDef {
    // 所有成员...
};
这样用户完全看不到内部成员，只能通过API函数操作。防止用户直接 Button.ClickCnt = 0 这种绕过接口的写法。
但这对平衡车项目来说过度设计了。原作者选择把所有成员暴露出来，好处是调试方便——你可以直接Watch结构体看所有状态。这是"工程实用"和"理论严谨"之间的合理折中。
---
六、Proc模式 — 轮询 vs 中断
按钮检测有两种方案：
 	中断方式
原理	引脚电平变化触发中断
响应速度	微秒级
代码复杂度	高（要处理中断嵌套、重入）
CPU占用	几乎为零（没按键就不进中断）
为什么平衡车选轮询？
1. 按键响应不需要微秒级精度，50ms的延迟人感觉不到
2. 主循环本身已经在高速轮询（<1ms一轮），轮询开销可以忽略
3. 轮询代码天然是非阻塞、可重入安全的，不存在中断竞争问题
4. 省中断资源——EXTI已经给编码器用了
老兵经验：嵌入式里有个原则——能用轮询解决的，不要用中断。中断是稀缺资源，带来的复杂度（优先级嵌套、数据竞争、调试困难）往往超过收益。只有真正硬实时的需求（编码器、电机过流保护、通信协议时序）才值得用中断。
---
七、防御性编程细节
回调调用前先判空
if(Button->button_long_pressed_cb)  // 先检查是否注册了回调
{
    Button->button_long_pressed_cb(Button->LongPressTicks);
}
Init() 里把所有回调初始化为0（NULL）：
Button->button_pressed_cb = 0;
Button->button_released_cb = 0;
// ...
如果用户没注册回调就触发事件，不会空指针崩溃——安全兜底。
LastPressedTime != 0 的防护
if(Button->LastPressedTime != 0 ...)
开机时 LastPressedTime=0，CurrentTime-0 会是一个很大的数，可能误触发长按。这个判断防止开机瞬间误判。
---
八、这个模板你可以直接复用
掌握了这个库的设计模式，你可以用同样的骨架封装任何外设驱动：
XXX.h  → 定义 XXX_TypeDef 结构体 + 声明 Init/Proc/SetCb 接口
XXX.c  → 实现 Init（配硬件）、Proc（状态机检测）、回调分发
app_xxx.c → 创建对象、注册回调、写业务逻辑

>
一个设计良好的库，应该让人只看接口就能用。这个Button库对外暴露了这些接口：
初始化：    My_Button_Init()         — 绑定引脚，初始化内部状态
设置回调：  My_Button_SetPressCb()   — 注册"按下"回调
            My_Button_SetReleaseCb() — 注册"松开"回调
            My_Button_SetClickCb()   — 注册"单击/连击"回调
            My_Button_SetLongPressCb()— 注册"长按"回调
运行：      My_Button_Proc()         — 在主循环中调用


```c
typedef struct {
    // === 配置参数（初始化时设置，之后不变）===
    GPIO_TypeDef *GPIOx;          // 接在哪个端口
    uint16_t GPIO_Pin;            // 接在哪个引脚
    void (*button_pressed_cb)(void);          // 按下回调
    void (*button_released_cb)(void);         // 松开回调
    void (*button_clicked_cb)(uint8_t clicks);// 连击回调
    void (*button_long_pressed_cb)(uint8_t ticks); // 长按回调
    uint32_t LongPressThreshold;  // 长按判定阈值（默认1000ms）
    uint32_t LongPressTickInterval;// 长按后持续触发间隔（默认100ms）
    uint32_t ClickInterval;       // 连击判定间隔（默认200ms）
    // === 运行时状态（库内部维护）===
    uint8_t  LastState;           // 上次按钮状态：0=松开，1=按下
    uint8_t  ChangePending;       // 消抖标志：1=正在等待消抖确认
    uint32_t PendingTime;         // 消抖起始时间
    uint32_t LastPressedTime;     // 上次按下的时刻
    uint32_t LastReleasedTime;    // 上次松开的时刻
    uint8_t  LongPressTicks;      // 长按已经触发了多少次tick
    uint32_t LastLongPressTickTime;// 上次长按tick的时间
    uint8_t  ClickCnt;            // 连击计数器
} Button_TypeDef;
```
- 我大体看一下他的逻辑，还是比较清晰的，但的确有深入的CPP面对对象的过程，大体上是初始化模块，Proc调用模块，参数配置模块，静态函数实现模块，回调函数映射模块，最主要的是理解它配置了哪些功能和这些功能实现的方法
- 结构体分为配置层和运行层，这个是不是可以再解析
- 实现了按下逻辑，松开逻辑，连击逻辑和长按逻辑，和时间参数的判定
- 本质上也是分为执行函数和标志位函数，Proc里面是进行标志和进行消抖，然后再调用执行逻辑


```c
#include "button.h"
#include "delay.h"

#define BUTTON_SETTLING_TIME             10   // 按钮消抖延迟
#define BUTTON_CLICK_INTERVAL            200  // 按钮多击时每次点击的时间最大时间间隔
#define BUTTON_LONG_PRESS_THRESHOLD      1000 // 按钮长按最小时间
#define BUTTON_LONG_PRESS_TICK_INTERNVAL 100  // 长按后持续触发的时间间隔

static void OnButtonPressed(Button_TypeDef *Button);
static void OnButtonReleased(Button_TypeDef *Button);
static void OnButtonEveryPolled(Button_TypeDef *Button, uint8_t State, uint32_t currentTime);
static void GPIOClockCmd(GPIO_TypeDef *GPIOx, uint8_t Enable);

// 
// @简介：用于初始化按钮的驱动
// @参数：Button - 按钮的名称
// @返回值：无
//
void My_Button_Init(Button_TypeDef *Button, Button_InitTypeDef *Button_InistStruct)
{
	Button->GPIOx = Button_InistStruct->GPIOx;
	Button->GPIO_Pin = Button_InistStruct->GPIO_Pin;
	Button->button_pressed_cb = 0;
	Button->button_released_cb = 0;
	Button->button_clicked_cb = 0;
	Button->button_long_pressed_cb = 0;
	Button->LongPressThreshold = BUTTON_LONG_PRESS_THRESHOLD;
	Button->ClickInterval = BUTTON_CLICK_INTERVAL;
	Button->LongPressTickInterval = BUTTON_LONG_PRESS_TICK_INTERNVAL;
	
	// #1. 使能GPIOx的时钟
	GPIOClockCmd(Button->GPIOx, 1);
	
	// #2. 初始化IO引脚
	GPIO_InitTypeDef gpio_init_struct;
	
	gpio_init_struct.GPIO_Pin = Button->GPIO_Pin;
	gpio_init_struct.GPIO_Mode = GPIO_Mode_IPU;
	
	GPIO_Init(Button->GPIOx, &gpio_init_struct);
	
	if(Button->LongPressThreshold == 0)
	{
		Button->LongPressThreshold = BUTTON_LONG_PRESS_THRESHOLD;
	}
	
	if(Button->LongPressTickInterval == 0)
	{
		Button->LongPressTickInterval = BUTTON_LONG_PRESS_TICK_INTERNVAL;
	}
	
	if(Button->ClickInterval == 0)
	{
		Button->ClickInterval = BUTTON_CLICK_INTERVAL;
	}
	
	Button->LastState = 0; // 初始状态下假设按钮是松开的
	Button->ChangePending = 0; 
	Button->PendingTime = 0;
	Button->LastPressedTime = 0;
	Button->LastReleasedTime = 0;
	Button->LongPressTicks = 0;
	Button->ClickCnt = 0;
}

// 
// @简介：按钮的进程函数
// @参数：Button - 按钮的名称
// @注意：该方法需要在main函数的while循环中调用
//
void My_Button_Proc(Button_TypeDef *Button)
{
	uint8_t currentState;
	
	uint32_t currentTime = GetTick(); // 获取当前时间
	
	// 按键消抖
	if(Button->ChangePending)
	{
		if (currentTime >= Button->PendingTime + BUTTON_SETTLING_TIME) // 已渡过按钮抖动时间
		{
			currentState = GPIO_ReadInputDataBit(Button->GPIOx, Button->GPIO_Pin) == Bit_RESET ? 1 : 0;
			
			if(currentState != Button->LastState)
			{
				if(currentState == 1) 
					OnButtonPressed(Button); // #1. 按钮按下
				else 
					OnButtonReleased(Button); // #2. 按钮松开
			}
			Button->LastState = currentState;
			Button->ChangePending = 0;
		}
	}
	else
	{
		currentState = GPIO_ReadInputDataBit(Button->GPIOx, Button->GPIO_Pin) == Bit_RESET ? 1 : 0;
		
		if(currentState != Button->LastState)
		{
			Button->PendingTime = currentTime;
			Button->ChangePending = 1;
		}
	}
	
	OnButtonEveryPolled(Button, Button->LastState, currentTime); // #3. 按钮状态被检测
}

//
// @简介：返回按钮的当前状态
//
// @返回值：0 - 按钮松开  1 - 按钮按下
//
uint8_t MyButton_GetState(Button_TypeDef *Button)
{
	return Button->LastState;
}

//
// @简介：设置按钮的单击间隔，短于此间隔视为连击
// @参数：Interval - 单击间隔（单位毫秒）
//
void My_Button_ClickIntervalConfig(Button_TypeDef *Button, uint32_t Interval)
{
	Button->ClickInterval = Interval;
}

void My_Button_LongPressConfig(Button_TypeDef *Button, uint32_t Throshold, uint32_t TickInterval)
{
	Button->LongPressThreshold = Throshold;
	Button->LongPressTickInterval = TickInterval;
}


void My_Button_SetLongPressCb(Button_TypeDef *Button, void (*LongPressCb)(uint8_t ticks))
{
	Button->button_long_pressed_cb = LongPressCb;
}

void My_Button_SetPressCb(Button_TypeDef *Button, void (*PressCb)(void))
{
	Button->button_pressed_cb = PressCb;
}

void My_Button_SetReleaseCb(Button_TypeDef *Button, void (*ReleaseCb)(void))
{
	Button->button_released_cb = ReleaseCb;
}

void My_Button_SetClickCb(Button_TypeDef *Button, void (*ClickCb)(uint8_t clicks))
{
	Button->button_clicked_cb = ClickCb;
}

//
// @简介：处理按钮按下的动作
//
static void OnButtonPressed(Button_TypeDef *Button)
{
	Button->LastPressedTime = GetTick();
	
	// 调用按钮按下的回调函数
	if(Button->button_pressed_cb != 0)
	{
		Button->button_pressed_cb();
	}
}

//
// @简介：处理按钮松开的动作
//
static void OnButtonReleased(Button_TypeDef *Button)
{
	Button->LastReleasedTime = GetTick();
	
	// 调用按钮松开的回调函数
	if(Button->button_released_cb != 0)
	{
		Button->button_released_cb();
	}
	
	// 松开后长按计数清零
	Button->LongPressTicks = 0;
	
	if(Button->LastReleasedTime - Button->LastPressedTime < Button->LongPressThreshold)
	{
		Button->ClickCnt++;
	}
	else
	{
		Button->ClickCnt = 0;
	}
}

//
// @简介：处理每一次按钮轮询的动作
//
static void OnButtonEveryPolled(Button_TypeDef *Button, uint8_t State, uint32_t CurrentTime)
{
	/* 处理按钮长按的动作 */
	
	if(Button->LastState == 1)
	{
		if(Button->LongPressTicks == 0) // 如果长按未被触发
		{
			if(Button->LastPressedTime!= 0 
				&& CurrentTime - Button->LastPressedTime > Button->LongPressThreshold) // 且已超过触发时间
			{
				Button->LongPressTicks = 1;
			
				if(Button->button_long_pressed_cb)
				{
					Button->button_long_pressed_cb(Button->LongPressTicks); // 触发长按回调函数
				}
				
				Button->LastLongPressTickTime = GetTick();
			}
		}
		else
		{
			if(CurrentTime - Button->LastLongPressTickTime > Button->LongPressTickInterval) // 超过Tick间隔
			{
				Button->LastLongPressTickTime = GetTick();
				
				Button->LongPressTicks++;
				
				if(Button->button_long_pressed_cb)
				{
					Button->button_long_pressed_cb(Button->LongPressTicks); // 触发长按回调函数
				}
			}
		}
	}
	
	/* 处理按钮连击动作 */
	
	if(Button->ClickCnt > 0 && Button->LastState == 0 && (GetTick() - Button->LastReleasedTime) > Button->ClickInterval)
	{
		if(Button->button_clicked_cb)
		{
			Button->button_clicked_cb(Button->ClickCnt);
		}
		
		Button->ClickCnt = 0; // 清除连击记录
	}
}

static void GPIOClockCmd(GPIO_TypeDef *GPIOx, uint8_t Enable)
{
	FunctionalState newState = Enable ? ENABLE : DISABLE;
	
	if(GPIOx == GPIOA)
	{
		RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, newState);
	}
	else if(GPIOx == GPIOB)
	{
		RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, newState);
	}
	else if(GPIOx == GPIOC)
	{
		RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, newState);
	}
	else if(GPIOx == GPIOD)
	{
		RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOD, newState);
	}
}
```

```c
#ifndef _BUTTON_H_
#define _BUTTON_H_

#include "stm32f10x.h"

typedef struct
{
	GPIO_TypeDef *GPIOx;   /* 按钮的端口号。
	                              可以取GPIOA..D中的一个*/
	
	uint16_t GPIO_Pin;     /* 按钮的引脚编号。
	                              可以取GPIO_Pin_0..15中的一个 */
} Button_InitTypeDef;

typedef struct 
{
	/* 初始化参数 */
	GPIO_TypeDef *GPIOx;
	uint16_t GPIO_Pin;
	void (*button_pressed_cb)(void);
	void (*button_released_cb)(void);
	void (*button_clicked_cb)(uint8_t clicks);
	void (*button_long_pressed_cb)(uint8_t ticks);
	uint32_t LongPressThreshold;
	uint32_t LongPressTickInterval;
	uint32_t ClickInterval; 
	
	uint8_t  LastState;     // 按钮上次的状态，0 - 松开，1 - 按下
	uint8_t  ChangePending; // 按钮的状态是否正在发生改变
	uint32_t PendingTime;   // 按钮状态开始变化的时间
	
	uint32_t LastPressedTime;  // 按钮上次按下的时间
	uint32_t LastReleasedTime; // 按钮上次松开的时间
	
	uint8_t LongPressTicks;
	uint32_t LastLongPressTickTime; 
	
	uint8_t ClickCnt;
	
} Button_TypeDef;

void My_Button_Init(Button_TypeDef *Button, Button_InitTypeDef *Button_InistStruct);
void My_Button_Proc(Button_TypeDef *Button);
uint8_t MyButton_GetState(Button_TypeDef *Button);

void My_Button_SetLongPressCb(Button_TypeDef *Button, void (*LongPressCb)(uint8_t ticks));
void My_Button_SetPressCb(Button_TypeDef *Button, void (*button_pressed_cb)(void));
void My_Button_SetReleaseCb(Button_TypeDef *Button, void (*button_released_cb)(void));
void My_Button_SetClickCb(Button_TypeDef *Button, void (*button_clicked_cb)(uint8_t clicks));

void My_Button_ClickIntervalConfig(Button_TypeDef *Button, uint32_t Interval);
void My_Button_LongPressConfig(Button_TypeDef *Button, uint32_t Throshold, uint32_t TickInterval);


#endif
```