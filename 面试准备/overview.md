一个基于 STM32F411CEU6 + FreeRTOS + LVGL v8.2 的开源智能手表,当前版本 V2.4.5。

一、整体架构

仓库根目录
├─ Firmware/          官方固件(bin/hex,可直接烧录)
├─ Software/
│  ├─ OV_Watch/       APP 主工程(Keil) —— 核心代码
│  └─ IAP_F411/       BootLoader 工程(Ymodem OTA 升级)
├─ Hardware/ 3D Modle/   硬件图纸与模型
└─ lv_sim_vscode_win/    LVGL PC 仿真工程
APP 与 Boot 分工:BootLoader 占 Flash 0x08000000 起 32KB,APP 偏移到 0x0800C000(main.c:75 设置 SCB->VTOR = 0x0000C000U)。Boot 启动时按住 KEY1 进 IAP 菜单,否则读 0x08008000 校验 "APP FLAG" 标记后跳转 APP(IAP_F411/Core/Src/main.c:133-195)。

二、启动流程与 13 个 FreeRTOS 任务
Core/Src/main.c → 系统时钟(HSI+PLL→100MHz)→ 外设初始化 → MX_FREERTOS_Init() 创建所有任务 → osKernelStart()。

所有任务在 User/Tasks/Src/user_TasksInit.c 中定义,协同关系如下:

任务	优先级	职责	通信
HardwareInitTask	High3	初始化所有硬件/传感器/LVGL,完成后自杀	—
WDOGFeedTask	High2	100ms 喂一次看门狗	—
StopEnterTask	High1	进入/退出 STOP 睡眠模式	Stop_MessageQueue
IdleEnterTask	High	息屏计时,超时先降背光再睡眠	Idle/IdleBreak 队列
KeyTask	Normal	扫描按键,决定回退/睡眠/前进	Key/Stop 队列
ScrRenewTask	Low1	按 Key 消息执行页面跳转	Key_MessageQueue
SensorDataUpdateTask	Low1	更新电量/步数/温湿度,按页面上报	HomeUpdata 队列
HRDataUpdateTask	Low1	心率页时采集 EM7028 算心率	—
ChargPageEnterTask	Low1	充电插入/拔出自动切充电页	事件标志 CHARG
MessageSendTask	Low1	解析 BLE(UART1)下发的命令	事件标志 UART
MPUCheckTask	Low2	300ms 判抬手/放下,放下则睡眠	Stop 队列
DataSaveTask	Low2	定时写 EEPROM(设置+当日步数)	DataSave 队列
LvHandlerTask	Low	跑 lv_task_handler(),喂 LVGL tick	—
线程间用 CMSIS-OS 消息队列(Key/Idle/Stop/IdleBreak/HomeUpdata/DataSave)+ 事件标志组(HardIntEventHandle)通信。

三、核心机制
1. 硬件中间层 HWDataAccess.c(最值得学习的移植设计)
全局结构体 HWInterface 用函数指针封装全部硬件:HWInterface.Power.Init()、HWInterface.IMU.GetSteps()…… UI 层完全不碰 HAL/寄存器。每个传感器带 ConnectionError 标志,初始化失败自动跳过。宏 HW_USE_HARDWARE 置 0 后 UI 可直接在 PC 仿真跑——这就是 README 说的"仿真/硬件无缝移植"。

2. 页面栈 PageManager.c

Page_t { void(*init)(), void(*deinit)(), lv_obj_t **page_obj }
PageStack_t { pages[MAX_DEPTH=6], top }
栈底永远是 HomePage,Page_Load(&Page_XXX) 入栈切页,Page_Back() 出栈回上一页,Page_Back_Bottom() 直接回首页。
坑(README 特别强调):入栈必须传 &ui_HomePage(指针的地址),因为 ui_HomePage 是动态变化的变量,直接传值会失效。
当前页用 Page_Get_NowPage() 获取,各任务据此判断"现在显示哪页"再决定采集逻辑。
3. 低功耗三段式
运行模式(70-80mA)→ 睡眠模式(STOP 模式,<1mA):

StopEnterTask 收到睡眠消息 → 串口 IO DeInit(省电)、LCD 关背光、触摸睡眠 → HAL_PWR_EnterSTOPMode
唤醒用 RTC 2 秒定时中断(main.c 中 HAL_RTCEx_SetWakeUpTimer_IT),醒来后用 MPU6050 加速度判断 MPU_isHorizontal()(roll/pitch ±0.5° 内=水平=抬手):抬手→继续运行;放下→goto sleep 再睡
关机模式:TPS63020 关闭使能,只有 Vbat→RTC 供电(Power_DisEnable())
4. 按键防误触设计(V2.4.1)
key.c 状态机:按下时不立即生效,松开后才返回键值——key_down 记录按下,key_up && !KEY1 && KEY2 状态翻转时才返回,配合 3ms 消抖。长按 KEY1 3 秒在 TIM1 中断里直接 Power_DisEnable() 关机。

5. EEPROM 数据存储(BL24C02)

[0x00:0x55][0x01:0xAA]  校验字节
[0x10]  抬手唤醒开关        [0x11]  APP同步开关
[0x20]  上次保存的日期       [0x21-0x22] 当日步数
DataSaveTask 每收到消息就把当前设置+步数写回,跨天自动清零步数重计。

6. 心率算法(自研峰值检测)
HrAlgorythm.c:官方库太慢被弃。用 7 点环形队列,当 data[3] 是局部极大值且与前一个峰间隔 >425ms(排除抖动)就记录一次心跳,60000/峰间隔 得瞬时心率,最后对 HR_List 7 个值取平均。

7. 计算器(双栈经典算法)
StrCalculate.c:NumSymSeparate() 解析表达式入栈(数字栈+符号栈),遇运算符先比优先级 SymisHighPriority,高优先级先 CalculateOne() 出栈运算,NumBehindPoint_Flag 专门处理小数点进位。

8. BLE 遥控协议
USART1 DMA 接收 + IDLE 中断 → 事件标志唤醒 MessageSendTask,支持命令:

OV → OK(握手)
OV+VERSION → 返回版本
OV+SEND → 上报时间/温湿/心率/血氧/步数
OV+ST=20230629125555 → 校时(需 APP 同步开关打开)
9. LCD 刷屏性能(V2.4.5 改进点)
lv_port_disp.c 用双缓冲(BUFFER_METHOD=2)+ LCD_Color_Fill() 走 SPI1+DMA 异步刷屏,DMA 完成回调 HAL_SPI_TxCpltCallback → lv_disp_flush_ready(),CPU 不用干等 SPI 传输。这就是 version.h 里说的 "LCD flush use synchronous DMA"。

四、UI 层(HomePage / MenuPage)
首页:大字体时钟、电量 Arc、步数 Bar、温/湿/心率三 Arc。每 500ms timer 检查数值是否变化,变了才刷 label(省电)。顶部下拉面板放 NFC/BLE/电源/设置/亮度滑条;右滑进入菜单。
菜单:13 个功能面板,每个用 #if XXX_PAGE_EN 宏控制是否编译,滚动位置存 ui_MenuScrollY 返回时恢复。
页面 tick 由 FreeRTOS TaskTickHook 里的 lv_tick_inc(1) 驱动,秒表计时也在这里累计。
五、关键代码位置速查
想找什么	去哪
系统初始化	Core/Src/main.c
任务定义与通信	User/Tasks/Src/user_TasksInit.c
睡眠/唤醒	user_RunModeTasks.c (StopEnterTask)
按键/页面跳转	BSP/KEY/key.c、user_KeyTask.c、user_ScrRenewTask.c
硬件中间层	User/Func/Src/HWDataAccess.c
页面栈	User/Func/Src/PageManager.c
心率算法	BSP/EM7028/HrAlgorythm.c
计算器	User/Func/Src/StrCalculate.c
数据存储	BSP/BL24C02/DataSave.c
BLE 协议	user_MessageSendTask.c
BootLoader	Software/IAP_F411/Core/Src/main.c + Ymodem/
版本信息	User/version.h
记忆文档已建好(memory/ 目录),