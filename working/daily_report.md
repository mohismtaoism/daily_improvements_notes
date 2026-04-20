# 2025.09.01

1.  Research CVSS tools for ultrasurgery
2.  Preparing LPC1114 pure-binary flashing solution and documents.

$$
\text{波特率} = \frac{\text{系统时钟频率}}{\text{分频系数}}
$$
# 2025.09.08

1. 早上赖床不想起，耗到了卡点来的公司；
2. 来了公司写了篇文章《机器猫与Rick & Morty的异曲同工之妙》，结果网页异常，没保存下来，我草泥马！
3. AC了一道简单Two points的167的题目，2-sum，主要是为了解决3-Sum。
4. 耀辉的主机资料到了，大体看了下代码，主控代码有两个，还有vol_pwm的代码，感觉功率板也有mcu，具体明天再说。
5. 尝试3-sum，失败了，思路是想固定一个数，然后套用2-sum；
6. 在群里跟个傻逼吵了一架；
7. 去十三陵水库游了600米，然后买了一只扒鸡；

# 2025.09.09

## 产品小需求
	如何用TTS技术做到不方便语音的时候还是以语音发信息

1. 早上来得还算早，四点多就醒了，刷手机刷到六点起来看早之前买的《Skunk Works》，看到七点多去上班；
2. 用AI分析了下耀辉的Main_ic11代码，形成了文件；
	1. 目前有两个串口，注释显示一个是与HMI通信，一个是与电压控制小板通信，多余的串口需要我们自己添加？
	2. Main_ic10，Main_ic10两个mcu，这两个mcu在整个系统是如何分工的？
	3. ModBus是基于串口实现的？是不是实现了三个mcu的一个组网通信？

# 2025.09.10

1. Building main_ic10 and Burning it to the main board.
2. Analyze main_ic10 source code.

# 2025.09.11

1. Build main_ic11 and burn it;
2. Verify the pedal's function.
Questions:
	1. 请提供屏幕供应商细节，并且诉我界面的美术资源怎么获取，比如那些警告界面？


# 2025.09.15

	1. Implementing and designing app_ui_protocol module to send the key information to Procision 1000;

# 2025.09.16
	1. Implementing Procision 1000's function to recieve data from serial port.
	2. Parsing the data from the ligasure board in Procision 1000.

# 2025.09.17
	1. Debugging the CRC16 function on MainFrame software.
	2. Update the CVSS report for BLSS.
> It's so sucks. The CRC16 function did not work correctly. The Calculating results from two methods are different, and the same method in different platform also leads to different results.

# 2025.09.18
	1. Debugging CRC16 modbus algorithm.

# 2025.09.22

$$
AP = \left( \frac{L_r I_{r_peak}^2}{B_{max} K_u J} \right)^{4/3} \quad (\text{cm}^4)
$$


$$
 N = \frac{L_r I_{r_peak}}{B_{max} A_e} \times 10^4
$$


$$
A_w = I_{r_rms}/J
$$

$$
P_{cu} = I_{r_rms}^2 \cdot R_{dc}
$$

$$
( P_{core} \approx k \cdot f^\alpha B_{max}^\beta \cdot V_e )
$$
1.  The Ligasure mainframe ic10 UART protocol implementation.
2.  The Ligasure mainframe ic10 UART protocol debugging.

# 2025.09.28


```bash
CertUtil -hashfile data_009_4253.tar MD5
CertUtil -hashfile data_009_4253.tar SHA1
CertUtil -hashfile data_009_4253.tar SHA256
CertUtil -hashfile data_014_4254.tar MD5
CertUtil -hashfile data_014_4254.tar SHA1
CertUtil -hashfile data_014_4254.tar SHA256
```

1. Technical support for the Brightness CVSS report.
2. Debugging Ligasure function on Procision 1000.
3. Try to optimize the RingWidget when activating the ligasure.

# 2025.11.17

1. Prepare software documents for Korea medical examination.
2. Push srcs to git server.
3. Analyze the pedal's release message issue.

# 2025.11.18

1. Answer questions from Korea teachers.
2. Reading the DHF files.

# 2025.11.19

1. Study DHF files;
2. Reading Insufflator source code.

# 2025.11.20


# 2025.11.21

# 2025.11.24

1. Travel to Changzhou;
2. Learning Insufflator source code architecture.
3. Learning source code from engineer from smartvein.

# 2025.11.25

1. Continue to learn source code.
2. Take part in Insufflator training.

# 2025.11.26

1. 学习气腹机恒压力pid调节控制流程；
2. 学习气腹机动态高压pid调节控制流程；
3. 气腹机耗材组件一致性测试；

> 那些回测 Backtrader`，或 `freqtrade类似这种开源

# 2025.11.27

1. 气腹机耗材组件一致性测试；
2. target参数数组标定；

# 2025.11.28

1. 参与与思脉得会议讨论接下来的工作；
2. 代码学习；

# 2025.12.01

1. New software version release;
2. Prepare software releasing documents;

# 2025.12.08

1. 软件发布文档处理；
2. ligasure 脚踏失效分析；

# 2025.12.09

1. NAS服务器调研；
2. ligasure 板子脚踏失效分析；
3. 宠物版手柄检测阻抗验证；

# 2025.12.10

1. NAS服务器调研；
2. Cyclone IV FPGA 学习；
3. ligasure 板子脚踏失效分析；

# 2025.12.11

1. ligasure 板子脚踏失效分析；
2. Cyclone IV FPGA 学习；
3. NAS服务器调研；

# 2025.12.12

1. ligasure 板子脚踏失效分析；
2. Cyclone IV FPGA 学习；
3. NAS服务器调研；

## X

1. Activate control.c OutputPower  这里面改能量减少：
	```c
	SetUS_Amp(real_amp / 4);
	```
2. Self-test control.c ScanFreq2
   ```c
   DACOutVol(45);
   ```
3. StartAutoFollow control.c
   change the activating frequence.

# 2025.12.15

# 2025.12.16

1. 组合逻辑电路在always块中用阻塞赋值"="；
2. 时序电路在always块中用非阻塞赋值"<=";

# 2025.12.22

1. Advantech Core Board malfunction analyzing;
2. Replace the Core Board for issued mainframe;
3. Ligaure source code ic11 analyze;

# 2025.12.23

1. Ligaure source code ic11 analyze;
2. Problematic mainframe issues analyzing;

# 2025.12.24

1. Do Pedal activating tests for 130nm, 230nm and 360nm scaples, and collecting logs;
2. Taking part in the broken scaple issues meeting.

# 2025.12.25

1. Analyze the linear interpolation algorithm about the real current.
	a. How to get the Current_Value table?
	b. Vk = ( 500 - G_ram->Control_parameter.Dac_ic11_out ) * 0.2; why calculate the voltage like this?
	c. get_needpower calculate the target power via the input the impedence.
	d. Run_power_Get
      	1. Using the desired voltage to calculate the impedence(*Rall*), then get the real impedence *Rwork* via Real_Impedance_Get
		2. Then using "Rwork" to get the target power(get_needpower).
2. Analyze the Pedal activating logs;

# 2025.12.26

1. Summary the activating algorithms flow chart.
2. System tests's impedence calculation tests.

# 2025.12.29

1. Making Procision 1000 algorithm's flow chart;
2. NAS Configuration for the Quality department;

# 2025.12.30

1. Assistanting Mr Huang to do void activating tests and collect failed logs;
2. Installing matlab on my laptop.
3. Assistanting Mr Zhu to do ssh commands.

# 2025.01.05

1. Trying to capture issued logs from some scapels with Taylor.
2. Prepare for beijing working trip.

# 2025.01.06

1. Travel to Beijing;
2. Verify the impedence calculation for verification team.

# 2025.01.07

1. Verify the impedence calculation for verification team.
2. Doing impedence tests to choose the best method;
3. Writing the Procision 1000 algorithm summary ppt;
4. Take part in the broken scapel meeting in headquart.

# 2026.01.07

代码逻辑在 traceFrequency 函数中。
核心方程：利用相位（Phi）和阻抗（R）通过多项式拟合计算频率偏差 $\Delta F$。
安全机制：如果计算出的频率 $F$ 偏离初始频率太远（超过 frequency_down_offset 或 frequency_up_offset），算法会判定为“失锁”，强制重置频率并把电流拉低 (pull_flat = true)，防止硬件损坏。

phase_offset 不是固定的。如果功率 $P_{avg} > 34$，Offset +1；如果 $P_{avg} < 32$，Offset -1。这是一种自适应机制，确保在不同负载下功率输出稳定。

峰值检测 (mFindPeak)：利用 RealTimeTrendChecker 检查功率趋势。当检测到切割完成（功率或阻抗出现特定峰值变化）后，算法会主动降低目标电流 (t_current)，以保护刀头和组织。

如果 $R_{ins} > 440$ 且 $|\Phi| > 35$，则判定为失谐，触发错误处理。


meeting

1. 代码点，与具体测试数据
2. 强生对比 何时升降
3. 标准测试环境：强生与普汇规格一样，一代二代杆
4. 异常，出厂自检log，检查一致性，阻抗，相位，阻抗异常是否都导致相位异常
z
软件，某个错误发生，记录关键log

01-06:48:53.680 [Warning]:[SystemTest::run:284]  i=0, code=0, min_imp=50.1726, freq=55460.5, cur=196


# 2026.01.08

1. Doing rfid tests for the new mainframe shell.
2. Doing ligasure current calibration tests;

# 2026.01.09

1. Trying to use printf via jlink wires.
2. Confirm the systemtest's impedence calculation solutions;
3. Moving stuffs to the new spaces.

# 2026.01.12

1. Prepare the hardwares and softwares for the ligasure current calibrations.
2. Calibrating the ligasure current table;

# 2026.01.13

1. Improve the detailed algorithm flowchart.
2. Take part in the broken scapels confrence with Headquater's leaders.

# 2026.01.14

1. Calibrating the ligasure current table;
2. Implementing the cutting logs chooing method.

# 2026.01.15

1. Prepare the Spanish temporary software release.
2. Calibrating the ligasure current table;

# 2026.01.16

1. Calibrating the ligasure current table;
2. Do new version software release.

# 2026.01.19

1. Calibrating the ligasure current table;
2. Calibrating the ligasure voltage table;

# 2026.01.20

1. Calibrating the ligasure impedance table;
2. Calibrating the ligasure voltage table;

# 2026.01.21

1. Put all things together to the mainframe.
2. Pusing source code into the git server.

# 2026.01.22

1. PA8,PB13 do PWM to drive the power circuits.
2. rfid module implementation.
3. Integrated mainframe test.

# 2026.01.23

1. RDID module implemantation
2.

# 2026.01.26

1. Implementing copying recent cutting logs and make related tar package.
2. Trouble shooting for NAS device's malfciuntion, it maybe wire's issue.

# 2026.01.27

1. Taking parting the broken scapel's meeting.
2. Setting up the envirnment for rfid tests.

# 2026.01.28

1. Implementing the app_rfid_queue module to recieve rfid data(Using ringbuffer internally);

# 2026.01.28

1. Designing a finite state machine to implement requesting the rfid card(including anti-collision, choosing chard, and the second anti-collision and choosing card)

# 2026.01.30

1. Continue to implemebnt this state machine.
2. Fix the ui's typos.

# 2026.02.02

1. The rfid board is reversed, line order from up to down: Yellow, Orange, Red, Blue.
2. The usb-serial port stick's chip side is showing: line order from up to down: Yellow, Orange, Red, Blue.

The response: size=8: 0x55 0x00 0x00 0x02 0x44 0x00 0x13 0x03
chiptype: 0x4400
The response: size=8: 0x55 0x00 0x00 0x02 0x44 0x00 0x13 0x03
The response: size=10: 0x55 0x00 0x00 0x04 0x88 0x04 0x56 0xce 0x45 0x03


The response: size=6: 0x55 0xe5 0x00 0x00 0xb0 0x03
The response: size=6: 0x55 0xe4 0x00 0x00 0xb1 0x03

0x55 0xe3 0x00 0x00 0xb6 0x03

I have finished the basic rfid module in stm32 dev board.

# 2026.02.03

1. Porting the rfid module to ic10;
2. Adding related codes in ultrasurgery;

# 2026.02.04

1. Finalize the copying recent blades data feature.
2.

eb90000100

eb 90 00 01 00 87 2c

5541 - 2006 = 3535

```c
		for(i=0; i<len; i++)
		{
			USART_SendData(USART2, buf[i]);
			while(USART_GetFlagStatus(USART2, USART_FLAG_TXE) == RESET )
			{
				j++;
				if(j > 5000)
				{
					break;
				}
			}
		}
```

# 2026.02.24

1. Debugging the rfid module for EMC pre-compliance tests.

# 2026.02.25

1. Split the serial port related codes from Ligasure UI.
2. Implement SerialPortManager class.

# 2026.02.26

1. Let the LigasureShow widget only do ui's works, and using signals from SerialPortManager to change ligasure ui.
2. Improve the cutting data files capturing mechanism.

# 2026.3.2

1. Finding a way to print debug information under the case of without serial port.
2. Calibrating the open circuit case and the short circuit case.

# 2026.3.3

After my diligent investigations, RTTE can use CMSIS dap burner to implement the printf function.

# 2026.3.4

1. Travel to Beijinig;
2. Optimizing the handle's quality method, ready to using Phases variance to detect them;

# 2026.3.5

1. Take part in the broken scapel's meeting;
2. Calibrating for ligasure's open circuit's judging conditions;
3. Summaring the ligasure algorithm;

# 2026.3.6

1. Calibrating the ligasure function successfully.

# 2026.3.9

1. Redesign the communication protocol between the ARM linux and the ic10.
2. Implement the ic10 side's communication.

# 2026.3.10

1. Calibrate the current table;
   a. Set the DAC to 350；
   b. Then set the different impedance values, and read the currnet value from the source, and also read the current value from electrisurgical analyzer. Total 15 pairs data.

2. Voltage table

Set 50 100 150 200 250 300 350 400 450

3. Impedance table

4. Recalibrate the number 2's mainframe.

# 2026.3.11

1. Implement the qt side's new communication's architecture.
2. Outline the open circuit error's corner cases.

# 2026.3.12

1. Debug the whole process between the arm linux and the ic10.
2. Let the rfid only read once during boot.

# 2026.3.13

1. Debug the rfid module.
2. Push all related source code into the git server.

# 2026.3.16

1. Potential changing languge issues analyzing;
2. Calibrating widget design.

# 2026.3.17

1. Calibrating widget design.
2. Implement SettingElectroSurgery widget.

# 2026.3.18

1. Designing the whole calibrating system;

# 2026.3.19

## Variance matrices

| Scenario | Variance of Phase |
| --- | --- |
| 一代+一体洪| 4708 |
| 一代+一体自制| 1569 |
| 一代+强生| 3126 |
| 一代+耀辉| 4027 |
| 一代+越安| 5070 |
| 二代+一体洪| 5945 |
| 二代+一自制| 4051 |
| 二代+强生| 4334 |
| 二代+耀辉| 4288 |
| 二代+越安| 5395 |

1. Prepare phase variance checking special softwares.
2. Do test cases with the specail softwares.
3. According to the blades saving feature, we will save recent 8 blades when the second detuned warning show.

# 2026.3.20

1. Implement the calibrating system in ic10 side.
2. Help Liuxia to set up the new computer wokring environment.

# 2026.3.23

1. Debug the recent blades data saving mechanism.
2. Implement and design the calibrating layer in ic10.

# 2026.3.24

## Tasks
1. Complete the ic10 layer;
2. Writing an essay about Qingzhao(2);
3. Start ad9838 project;

# Actual achievements

1. Complete the ic10 layer of the calibrating system;
2. Help the manufacturing department to analyze the impedance issue of the mainframe.

# 2026.3.25

In IC11, ModbusTask_Marster has the state machine to recieve and send the data.

rR8Code_Marster to recieve from ic10

Implement the current feeddback mechanism.

# 2026.3.26

1. Implement ESBasicDACWidget.

# 2026.3.27

1. Configure staffs' NAS permissions;
2. Implement ESBasicDACWidget.

# 2026.4.1

1. Current collecting routine
	The *I* is 12 bits unsigned integer from FPGA's da chip.
	```c
	float cur_avg=I*2.5/4096;
	Iprms=cur_avg/2.965;
	```
2. Optimizing ESBasicDACWidget;

# 2026.4.2

Optimizing ESBasicDACWidget;

# 2026.4.3

# 2026.04.08

# 2026.4.13

1. Do phase variance verification for 10 scapels.
2. Pedal activating issue tracking;

| Number | Variance of Phase | Result |
| --- | --- | --- |
| 1 | 3137 | Failed |
| 2 | 4140 | OK |
| 3 | 4200 | ok |
| 4 | 4334 | ok |
| 5 | 3134 | Failed |
| 6 | 3399 | Failed |
| 7 | 4417 | ok |
| 8 | 3864 | ok |
| 9 | 3635 | ok |
| 10 | 3635 | ok |

# 2026.4.14

1. Taking part in the broken scapel analysis meeting.
2. Pedal activating issue tracking;

# 2026.4.15

Pedal activating issue tracking;

# 2026.04.16

1. Help yifan fetch the components.
2. Debug the whole calibrating system.

## Number 1

```c
const float Current_Value [2][15] = {
    {55,   80,   122,  140,  263,  327,  431,  511,  629,  825,  1181, 1811, 2040, 2393, 3550},  //内部计数值
    {270, 274,   287,  296,  366,  404,  467,  517,  589,  709,  926,  1261, 1426, 1630, 2286}   //实际电流值
};

const float Impedence_Value [2][15] = {
    {61, 88, 122, 146, 165, 178, 189, 203, 213, 221, 235, 242, 264, 283, 300},
    {10, 25,  50,  75,  100, 125, 150, 200, 250, 300, 500, 800, 1000, 1500, 2000}
};

const float voltage_factory = 0.28;
```

## Number 2

```c
const float Current_Value [2][15] =
{
    {52, 98, 141, 167, 221, 265, 327, 431, 511, 628, 819, 1162, 1976, 2291,  3253},
    {246, 250, 252, 255, 268, 289, 323, 380, 423, 489, 593, 782, 1224, 1384, 1850}
};

const float Impedence_Value [2][15] = {
    {83, 117, 165, 200, 226, 247, 263, 288, 302, 312, 317, 318, 351,  372,  390},
    {10, 25,  50,  75,  100, 125, 150, 200, 250, 300, 500, 800, 1000, 1500, 2000}
};

const float voltage_factory = 0.28;
```

# 2026.04.17

1. Taking part in the CE file preparing meeting for energy platform.
2. Debugging the calibrating routine;