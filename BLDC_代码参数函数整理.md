# BLDC 项目代码整理（参数 + 函数）

> 范围：基于仓库内你提供的 8 个 `.txt` 文件（由主要 `.c` 文件提取）。
> 目标：按**系统功能**整理参数与函数，标注来源文件，方便你边读代码边查表。
> 说明：本版先完成“核心可运行链路 + 关键控制模块”的完整表格；如果你认可结构，我下一步可以继续细化到“结构体字段级别逐项解释（接近逐行字典）”。

---

## 0) 8个代码阅读顺序（最简版）

1. `main.txt`（看硬件初始化入口）
2. `mc_parameters.txt`（看所有控制参数实例）
3. `mc_config.txt`（看控制对象装配关系）
4. `stm32g4xx_mc_it.c.txt`（看中断如何触发控制）
5. `mc_tasks.txt`（看状态机与FOC调度主流程）
6. `r3_2_g4xx_pwm_curr_fdbk.txt`（看PWM与相电流采样细节）
7. `hall_speed_pos_fdbk.txt`（看霍尔角度/速度反馈）
8. `pid_regulator.txt`（最后看PI/PID算法细节）

> 一句话记忆：**先“框架和参数”，再“中断与任务”，最后“驱动与算法细节”**。

---

## 1) 系统功能分层（先看这个）

| 层级 | 功能 | 主要文件 |
|---|---|---|
| 启动与外设层 | 时钟、GPIO、DMA、ADC、TIM、UART 初始化 | `main.txt` |
| 电机控制调度层 | 启动 MC 内核、状态机、中频/高频任务、安全任务 | `mc_tasks.txt` |
| 实时中断层 | ADC/TIM/霍尔/UART/SysTick 中断服务 | `stm32g4xx_mc_it.c.txt` |
| 功率驱动层 | 三电阻采样、电流重构、PWM 开关、保护/故障处理 | `r3_2_g4xx_pwm_curr_fdbk.txt` |
| 速度位置反馈层 | 霍尔测速、角度估算、平均速度计算 | `hall_speed_pos_fdbk.txt` |
| 控制算法层 | PI/PID 参数管理与执行 | `pid_regulator.txt` |
| 参数装配层 | 各类句柄/参数实例化与绑定 | `mc_parameters.txt`, `mc_config.txt` |

---

## 2) 参数总表（按系统功能、含重要度）

> 重要度：
> - **A** = 直接影响电机安全/闭环稳定/主流程
> - **B** = 影响性能与行为
> - **C** = 辅助常量或内部状态

### 2.1 启动与外设层（`main.txt`）

| 参数/变量 | 含义 | 重要度 | 文件 |
|---|---|---|---|
| `hadc1`, `hadc2` | 双 ADC 句柄，供相电流/母线电压等采样 | A | `main.txt` |
| `hcordic` | CORDIC 外设句柄（常用于快速三角运算） | B | `main.txt` |
| `hopamp1`, `hopamp2` | 运放句柄，用于电流采样前端 | A | `main.txt` |
| `htim1` | 主 PWM 定时器（电机驱动核心） | A | `main.txt` |
| `htim4` | 霍尔测速相关定时器 | A | `main.txt` |
| `huart3` + `hdma_usart3_rx/tx` | 串口通信与 DMA | B | `main.txt` |

### 2.2 调度与状态机层（`mc_tasks.txt`）

| 参数/宏 | 含义 | 重要度 | 文件 |
|---|---|---|---|
| `CHARGE_BOOT_CAP_MS(_2)` | 预充自举电容等待时间 | A | `mc_tasks.txt` |
| `OFFCALIBRWAIT_MS(_2)` | 关断校准等待时间 | B | `mc_tasks.txt` |
| `STOPPERMANENCY_MS(_2)` | 停机保持时间（防抖/保护） | A | `mc_tasks.txt` |
| `CHARGE_BOOT_CAP_TICKS(_2)` | 预充等待 tick 值 | A | `mc_tasks.txt` |
| `OFFCALIBRWAITTICKS(_2)` | 校准等待 tick 值 | B | `mc_tasks.txt` |
| `STOPPERMANENCY_TICKS(_2)` | 停机保持 tick 值 | A | `mc_tasks.txt` |
| `VBUS_TEMP_ERR_MASK` | 电压/温度故障掩码组合 | A | `mc_tasks.txt` |
| `FOCVars[]` | 每电机 FOC 运行态变量 | A | `mc_tasks.txt` |
| `pwmcHandle[]` | PWM/电流反馈组件句柄数组 | A | `mc_tasks.txt` |
| `pCLM[]` | 电压矢量圆限幅句柄 | A | `mc_tasks.txt` |
| `pREMNG[]` | 电流给定斜坡管理器 | B | `mc_tasks.txt` |
| `hMFTaskCounterM1` | M1 中频任务计数器 | B | `mc_tasks.txt` |
| `hBootCapDelayCounterM1` | M1 自举电容延时计数 | A | `mc_tasks.txt` |
| `hStopPermanencyCounterM1` | M1 停机保持计数 | A | `mc_tasks.txt` |
| `bMCBootCompleted` | MC 内核是否启动完成标志 | A | `mc_tasks.txt` |

### 2.3 中断与时间基准层（`stm32g4xx_mc_it.c.txt`）

| 参数/宏 | 含义 | 重要度 | 文件 |
|---|---|---|---|
| `SYSTICK_DIVIDER` | SysTick 到 ms 的分频换算 | A | `stm32g4xx_mc_it.c.txt` |

### 2.4 霍尔反馈层（`hall_speed_pos_fdbk.txt`）

| 参数/宏 | 含义 | 重要度 | 文件 |
|---|---|---|---|
| `LOW_RES_THRESHOLD` | 触发“降预分频”阈值（测速分辨率策略） | B | `hall_speed_pos_fdbk.txt` |
| `HALL_COUNTER_RESET` | 霍尔计数器复位值 | C | `hall_speed_pos_fdbk.txt` |
| `S16_120_PHASE_SHIFT` / `S16_60_PHASE_SHIFT` | 电角度相位偏移（120°/60°） | A | `hall_speed_pos_fdbk.txt` |
| `STATE_0 ... STATE_7` | 霍尔状态机状态编码 | A | `hall_speed_pos_fdbk.txt` |
| `NEGATIVE` / `POSITIVE` | 转向符号常量 | B | `hall_speed_pos_fdbk.txt` |
| `HALL_MAX_PSEUDO_SPEED` | 最大伪速度限幅 | A | `hall_speed_pos_fdbk.txt` |
| `CCER_CC1E_Set/Reset` | 定时器捕获使能位操作掩码 | B | `hall_speed_pos_fdbk.txt` |

### 2.5 PWM+电流采样层（`r3_2_g4xx_pwm_curr_fdbk.txt`）

| 参数/宏 | 含义 | 重要度 | 文件 |
|---|---|---|---|
| `TIMxCCER_MASK_CH123` | TIM CH1/2/3 及互补通道使能掩码 | A | `r3_2_g4xx_pwm_curr_fdbk.txt` |

### 2.6 项目参数装配层（`mc_parameters.txt`, `mc_config.txt`）

| 参数/实例 | 含义 | 重要度 | 文件 |
|---|---|---|---|
| `FREQ_RATIO` / `FREQ_RELATION` | 单电机场景下的频率关系占位配置 | B | `mc_parameters.txt`, `mc_config.txt` |
| `R3_3_OPAMPParamsM1` | M1 运放拓扑与输入映射配置 | A | `mc_parameters.txt` |
| `R3_2_ParamsM1` | M1 三电阻采样+PWM 底层参数总成 | A | `mc_parameters.txt` |
| `PIDSpeedHandle_M1` | 速度环 PID 参数实例 | A | `mc_parameters.txt` |
| `PIDIqHandle_M1` / `PIDIdHandle_M1` | q/d 电流环 PI 参数实例 | A | `mc_parameters.txt` |
| `STO_Handle`（若有）/ `HALL_M1` | 转速位置反馈实例（本项目为 Hall） | A | `mc_parameters.txt` |
| `SpeednTorqCtrlM1` | 速度-转矩控制器参数 | A | `mc_parameters.txt` |
| `PWM_Handle_M1` | PWM 电流反馈运行句柄 | A | `mc_parameters.txt` |
| `TempSensor_M1` | 温度传感器（虚拟）参数 | B | `mc_parameters.txt` |
| `BusVoltageSensor_M1` + `RealBusVoltageSensorFilterBufferM1[]` | 母线电压检测与滤波 | A | `mc_parameters.txt` |
| `RampExtMngrHFParamsM1` | 高频斜坡管理参数 | B | `mc_parameters.txt` |
| `CircleLimitationM1` | 电压矢量圆限幅参数 | A | `mc_parameters.txt` |
| `Mci[]`, `pSTC[]`, `pPIDIq[]`, `pPIDId[]`, `pTemperatureSensor[]`, `pMPM[]` | 控制对象索引与关联表 | A | `mc_parameters.txt` |

---

## 3) 函数总表（按系统功能，含优先级）

### 3.1 启动与硬件初始化（`main.txt`）

| 函数 | 作用 | 优先级 |
|---|---|---|
| `main` | 主入口，HAL/时钟/外设初始化并进入主循环 | A |
| `SystemClock_Config` | 系统时钟树配置 | A |
| `MX_GPIO_Init` | GPIO 初始化 | A |
| `MX_DMA_Init` | DMA 初始化 | B |
| `MX_ADC1_Init`, `MX_ADC2_Init` | ADC 初始化 | A |
| `MX_CORDIC_Init` | CORDIC 初始化 | C |
| `MX_OPAMP1_Init`, `MX_OPAMP2_Init` | 运放初始化 | A |
| `MX_TIM1_Init`, `MX_TIM4_Init` | PWM/测速定时器初始化 | A |
| `MX_USART3_UART_Init` | 串口初始化 | B |
| `MX_NVIC_Init` | 中断优先级与使能 | A |
| `Error_Handler` | 错误兜底处理 | A |
| `assert_failed` | 断言失败回调 | C |

### 3.2 电机任务与状态机（`mc_tasks.txt`）

| 函数 | 作用 | 优先级 |
|---|---|---|
| `MCboot` | 电机控制内核全局初始化 | A |
| `MC_RunMotorControlTasks` | 周期调度入口（中频+安全任务） | A |
| `MC_Scheduler` | 中频任务调度器 | A |
| `TSK_MediumFrequencyTaskM1` | M1 中频状态机主处理 | A |
| `TSK_HighFrequencyTask` | 高频 FOC 执行入口（在 ADC 中断中调用） | A |
| `FOC_CurrControllerM1` | M1 电流环控制器核心计算 | A |
| `FOC_Clear` | 清空 FOC 运行状态 | A |
| `FOC_CalcCurrRef` | 计算电流参考值（Id/Iq） | A |
| `FOC_InitAdditionalMethods` | 附加方法初始化 | B |
| `TSK_MF_StopProcessing` | 停机流程处理 | A |
| `TSK_SafetyTask`, `TSK_SafetyTask_PWMOFF` | 安全检查与关 PWM 处理 | A |
| `TSK_SetChargeBootCapDelayM1` / `TSK_ChargeBootCapDelayHasElapsedM1` | 自举预充计时管理 | A |
| `TSK_SetStopPermanencyTimeM1` / `TSK_StopPermanencyTimeHasElapsedM1` | 停机保持计时管理 | A |
| `GetMCI` | 获取电机控制接口对象 | B |
| `TSK_HardwareFaultTask` | 硬件故障任务 | A |
| `UI_HandleStartStopButton_cb` | 启停交互回调 | C |
| `mc_lock_pins` | 与引脚保护/锁定相关 | C |

### 3.3 中断服务（`stm32g4xx_mc_it.c.txt`）

| 函数 | 作用 | 优先级 |
|---|---|---|
| `ADC1_2_IRQHandler` | ADC 注入转换中断；触发高频 FOC | A |
| `TIMx_UP_M1_IRQHandler` | PWM 更新中断 | A |
| `TIMx_BRK_M1_IRQHandler` | 刹车/过流等 break 中断 | A |
| `SPD_TIM_M1_IRQHandler` | 霍尔测速定时器中断 | A |
| `MCP_RX_IRQHandler_A`, `USARTA_IRQHandler` | 串口协议接收中断 | B |
| `SysTick_Handler` | 系统节拍中断（调度入口） | A |
| `HardFault_Handler` | 硬 fault 异常处理 | A |
| `EXTI15_10_IRQHandler` | 外部中断线处理（按键/IO 触发等） | B |

### 3.4 霍尔测速与角度估算（`hall_speed_pos_fdbk.txt`）

| 函数 | 作用 | 优先级 |
|---|---|---|
| `HALL_Init` | 霍尔模块初始化 | A |
| `HALL_Clear` | 霍尔状态清空 | B |
| `HALL_CalcElAngle` | 计算电角度 | A |
| `HALL_CalcAvrgMecSpeedUnit` | 计算平均机械速度 | A |
| `HALL_SetMecAngle` | 设置机械角 | B |
| `HALL_Init_Electrical_Angle`(static) | 初始化电角度偏置 | B |

### 3.5 PI/PID 调节器（`pid_regulator.txt`）

| 函数 | 作用 | 优先级 |
|---|---|---|
| `PID_HandleInit` | PID 句柄初始化 | A |
| `PID_SetKP/KI/KD` | 设置比例/积分/微分增益 | A |
| `PID_GetKP/KI/KD` | 读取当前增益 | B |
| `PID_GetDefaultKP/KI` | 读取默认增益 | C |
| `PID_SetIntegralTerm` | 设置积分状态（支持预置/复位） | A |
| `PID_SetLower/UpperIntegralTermLimit` | 积分限幅 | A |
| `PID_SetLower/UpperOutputLimit` | 输出限幅 | A |
| `PID_SetPrevError` | 设置上次误差（D 项相关） | B |
| `PID_Get*Divisor`, `PID_Set*DivisorPOW2` | 增益缩放分母管理（定点实现） | B |
| `PI_Controller` | PI 控制计算 | A |
| `PID_Controller` | PID 控制计算 | A |

### 3.6 三电阻电流采样 + PWM 驱动（`r3_2_g4xx_pwm_curr_fdbk.txt`）

| 函数 | 作用 | 优先级 |
|---|---|---|
| `R3_2_Init` | 采样与 PWM 模块初始化 | A |
| `R3_2_CurrentReadingPolarization` | 电流零偏校准 | A |
| `R3_2_GetPhaseCurrents` / `_OVM` | 相电流重构读取（普通/OVM） | A |
| `R3_2_SetADCSampPointPolarization` | 校准时采样点设置 | A |
| `R3_2_SetADCSampPointSectX` / `_OVM` | 各扇区注入采样时刻设置 | A |
| `R3_2_WriteTIMRegisters`(inline) | 写入定时器比较寄存器 | A |
| `R3_2_TurnOnLowSides` | 低桥导通（预充/上电相关） | A |
| `R3_2_SwitchOnPWM` / `R3_2_SwitchOffPWM` | PWM 开关控制 | A |
| `R3_2_IsOverCurrentOccurred` | 过流状态检测 | A |
| `R3_2_RLDetectionModeEnable/Disable/SetDuty` | R/L 检测模式开关与占空比设置 | B |
| `RLTurnOnLowSidesAndStart` | R/L 测试模式启动辅助 | B |
| `R3_2_ADCxInit`, `R3_2_TIMxInit`(static) | ADC/TIM 底层初始化 | B |
| `R3_2_HFCurrentsPolarizationAB/C`(static) | 高频偏置采样内部函数 | B |
| `R3_2_RLGetPhaseCurrents`(static) | R/L 检测电流读取 | B |
| `R3_2_RLTurnOnLowSides`, `R3_2_RLSwitchOnPWM`(static) | R/L 检测驱动过程控制 | B |
| `R3_2_SetAOReferenceVoltage`(static) | DAC 比较阈值设定（保护相关） | B |

---

## 4) 建议你看代码的顺序（高效率）

1. 先读 `main` 和 `MCboot`，看“初始化链路”。
2. 再读 `ADC1_2_IRQHandler -> TSK_HighFrequencyTask -> FOC_CurrControllerM1`，看“实时控制主链”。
3. 再读 `TSK_MediumFrequencyTaskM1`，看“状态机和速度环”。
4. 最后分别深入 `R3_2_*`（驱动采样细节）、`HALL_*`（测速角度）、`PID_*`（控制器细节）。

---

## 5) 下一步我可以继续做什么（你说“继续”我就接着做）

- 继续做**第二版**：把 `mc_parameters.txt` 里每个结构体字段（例如 `R3_2_ParamsM1` 里的 `Tafter/Tbefore/Tsampling/Tdead` 等）逐项解释成“参数词典”。
- 继续做**第三版**：给每个函数补“输入参数/输出/副作用/被谁调用”。
- 继续做**第四版**：额外画一张“调用关系图（文本版）”，从中断到 FOC 核心一路串起来。


---

## 6) 结合你这张原理图 + 标准FOC框图：电机“按某速度稳定转动”到底怎么跑起来

> 面向初学者，先用“模块对照 + 时间线”理解。

### 6.1 原理图模块 与 代码模块一一对应

1. **MCU/时钟/中断（原理图左上 MCU/LED/按键）**
   - 代码入口是 `main()`，先 `HAL_Init()`，再 `SystemClock_Config()`，再初始化 GPIO/DMA/ADC/OPAMP/TIM/UART，最后 `MX_MotorControl_Init()` + `MX_NVIC_Init()`。
   - 这一步的意义：把“控制大脑”和“实时触发机制”先搭好。

2. **驱动 + 三相全桥（原理图右上 三相全桥，右中 驱动）**
   - 对应 `R3_2_*` 模块：`R3_2_Init()`、`R3_2_SwitchOnPWM()`、`R3_2_SwitchOffPWM()`。
   - 这一步的意义：能把 MCU 算出来的电压矢量，真正变成三相桥臂 PWM 脉冲。

3. **三相电流采样（原理图左下 运放/三相电流采样）**
   - 对应 `R3_2_CurrentReadingPolarization()`（先做零偏校准）和 `R3_2_GetPhaseCurrents()`（运行中取 Ia/Ib）。
   - 这一步的意义：FOC 必须闭环，先“测到电流”才能控电流。

4. **霍尔编码器（原理图下方 霍尔编码器）**
   - 对应 `HALL_Init()`、`HALL_CalcElAngle()`、`HALL_CalcAvrgMecSpeedUnit()`。
   - 这一步的意义：给 FOC 提供转子角度/速度，决定 Park/反Park变换是否对齐。

5. **母线电压（原理图右下 母线电压）**
   - 对应 `RVBS_Init(&BusVoltageSensor_M1)` 与安全任务 `TSK_SafetyTask()`。
   - 这一步的意义：过压/欠压保护，避免炸管。

---

### 6.2 “电机以目标速度转动”执行时间线（你可以按这个断点调试）

1. **上电初始化阶段**
   - `main()` 完成所有外设初始化。
   - `MCboot()` 完成电机控制对象初始化：PWM采样、霍尔、PID、母线电压、温度、MCI。

2. **启动命令进入状态机**
   - 状态机在 `TSK_MediumFrequencyTaskM1()` 内推进（IDLE -> 校准 -> 充电自举 -> RUN）。
   - 其中会做电流偏置校准、低边导通充 bootstrap 电容、打开 PWM。

3. **中断驱动的实时闭环开始**
   - ADC 注入中断 `ADC1_2_IRQHandler()` 每个 PWM 周期触发一次高频任务 `TSK_HighFrequencyTask()`。
   - 高频任务里先更新角度 `HALL_CalcElAngle()`，然后进入 `FOC_CurrControllerM1()`。

4. **FOC核心计算（每个 PWM 周期都做）**
   - 采样相电流 -> Clarke -> Park。
   - 误差 `Iq_ref-Iq`、`Id_ref-Id` 进 PI，得到 `Vq/Vd`。
   - 限幅后反 Park，最后 `PWMC_SetPhaseVoltage()` 更新三相 PWM 占空比。

5. **中频速度环慢速调节（比如 1kHz）**
   - `MC_Scheduler()` 周期调用 `TSK_MediumFrequencyTaskM1()`。
   - 速度控制器（`STC + PIDSpeed`）根据目标速度和霍尔反馈，慢慢调整 `Iq_ref`，实现“追到目标速度并稳住”。

---

### 6.3 想改成“某个目标速度稳定转”，你优先改哪里

> 原则：**先改目标，再改环路；先保守，再提性能。**

1. **先改默认目标速度（最直接）**
   - 在 `SpeednTorqCtrlM1` 里改 `MecSpeedRefUnitDefault`（宏来自 `DEFAULT_TARGET_SPEED_UNIT`）。
   - 同时确认最大/最小允许速度范围 `MaxAppPositiveMecSpeedUnit / MinAppPositiveMecSpeedUnit` 合理。

2. **再改速度环 PID（是否能稳）**
   - `PIDSpeedHandle_M1` 的 `hDefKpGain`、`hDefKiGain` 决定速度响应快慢和稳态误差。
   - 初学者建议：先小 `Kp`、小 `Ki` 能转稳，再逐步加快响应。

3. **电流环一般先不大改**
   - `PIDIqHandle_M1` / `PIDIdHandle_M1` 是内环，通常先保持 Workbench 默认。
   - 如果速度环怎么调都抖，再检查电流环和采样噪声。

4. **确认霍尔相位与极对数**
   - `HALL_M1` 里的 `bElToMecRatio(POLE_PAIR_NUM)`、`PhaseShift`、`SensorPlacement` 配错会出现“能转但抖/反转/效率低”。

5. **保护阈值别乱放开**
   - `DAC_OCP_Threshold`、`DAC_OVP_Threshold`、以及安全任务相关阈值建议先保守。

---

### 6.4 给初学者的最小改参步骤（实操）

1. 只改 `DEFAULT_TARGET_SPEED_UNIT`，下载运行，确认能启动并达到速度。
2. 若到不了速度：小步增加 `PID_SPEED_KP_DEFAULT`。
3. 若有静差：小步增加 `PID_SPEED_KI_DEFAULT`。
4. 若振荡/啸叫：先减小 `Kp`，再减小 `Ki`。
5. 若低速抖动明显：重点复查霍尔相位、采样偏置校准、布线噪声与接地。

---

### 6.5 一句话总结

**这套代码是“中频状态机管流程 + 高频中断跑FOC + 霍尔给角度 + 三相采样做电流闭环 + 速度环给Iq参考”的标准结构。**


---

## 7) 直接让电机按“某个速度”转：最小改代码方案

> 目标：上电后不依赖上位机，直接让电机转到你指定速度。

### 7.1 方案A（最稳妥，少改代码）：改默认目标速度 + 发送启动命令

1. 在参数里把默认目标速度改成你要的值：
   - `SpeednTorqCtrlM1.MecSpeedRefUnitDefault`
   - 对应宏一般是 `DEFAULT_TARGET_SPEED_UNIT`。
2. 保证范围允许：
   - `MaxAppPositiveMecSpeedUnit`
   - `MinAppPositiveMecSpeedUnit`
3. 在初始化后（`main` 的 `USER CODE BEGIN 2` 或主循环里一次性）发送启动命令。

> 这种方式改动最小，最不容易破坏现有状态机。

### 7.2 方案B（代码里直接下发速度命令）：推荐写法

在 `main` 里初始化完成后加“一次性启动”逻辑（示例）：

```c
/* USER CODE BEGIN PV */
static uint8_t auto_start_done = 0;
/* USER CODE END PV */

/* USER CODE BEGIN 2 */
// 可选：先给一个速度斜坡命令（单位是 MecSpeedUnit，不一定是 rpm）
// MCI_ExecSpeedRamp(&Mci[M1], TARGET_SPEED_UNIT, 1000);
/* USER CODE END 2 */

while (1)
{
  /* USER CODE BEGIN 3 */
  if (auto_start_done == 0U)
  {
    // 1) 启动电机
    MCI_StartMotor(&Mci[M1]);

    // 2) 给速度斜坡到目标值（例如 1000ms）
    MCI_ExecSpeedRamp(&Mci[M1], TARGET_SPEED_UNIT, 1000);

    auto_start_done = 1U;
  }
  /* USER CODE END 3 */
}
```

说明：
- `MCI_ExecSpeedRamp(...)` 在工程里已用于给速度控制器下发目标（`MCboot` 里就有一次默认调用）。
- 真正闭环跟踪目标速度仍由中频/高频任务自动完成，不需要你手写 FOC 算法。

### 7.3 你必须一起检查的3件事（否则“给了速度也不稳”）

1. **霍尔参数正确**：极对数、相位偏置、安装方向（否则速度环会抖动或反转）。
2. **速度环参数别太激进**：`PIDSpeed` 先小 `Kp/Ki`，再逐步加。
3. **电流采样已校准**：启动流程必须走 offset 校准 + bootstrap 充电（状态机会做）。

### 7.4 调试建议（一步一步）

1. 先把 `TARGET_SPEED_UNIT` 设低速值（比如额定的 10%~20%）。
2. 能稳定启动后，再逐步提高目标速度。
3. 若抖动：先降 `PID_SPEED_KP_DEFAULT`，再降 `PID_SPEED_KI_DEFAULT`。
4. 若上不去：先看母线电压/限流，再小步增 `Kp`。

