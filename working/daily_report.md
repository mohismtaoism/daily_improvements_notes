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

# 2026.04.20

 1. Setting up the local integrated mainframe working envirnment.
 2. Debugging the rfid module(Failed).
 3. Debugging the ic10 calibrating module;
 4. Debugging the Basic DAC setting and feedback widget.
 5. n8n

# 2026.04.21

https://www.tsinghua.org.cn/info/1954/15030.htm
https://www.tsinghua.org.cn/info/1952/17679.htm

1. Implement the Electrosurgery Biomed widget;
2. Debug the open circuit error cases;
3. Check the differences between the old audio chip and the new one.

# 2026.04.22

```cpp
QByteArray data;
data.append(dataArray[4]);
data.append(dataArray[5]);
data.append(dataArray[6]);
data.append(dataArray[7]);
data.append(dataArray[8]);
```

```bash
docker run -it --rm \
 --name n8n \
 -p 5678:5678 \
 -e GENERIC_TIMEZONE="Asia/Shanghai" \
 -e TZ="Asia/Shanghai" \
 -e N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true \
 -e N8N_RUNNERS_ENABLED=true \
 -v n8n_data:/home/node/.n8n \
 docker.n8n.io/n8nio/n8n
```

1. The new audio chip issues analyzing;
2. The Electro Surgery Biomed widget implementation.

# 2026.04.23

```bash
export PATH=/the/file/path:$PATH
```
1. solve the open circuit

# 2026.04.24

1. Implement ESCalibratingVoltage;
2. Verify 8 pedals' function;

```bash
sudo docker run -it --rm  --name n8n  -p 5678:5678  -e GENERIC_TIMEZONE="Asia/Shanghai"  -e TZ="Asia/Shanghai"  -e N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true  -e N8N_RUNNERS_ENABLED=true -e N8N_SECURE_COOKIE=false  -v n8n_data:/home/node/.n8n  docker.n8n.io/n8nio/n8n
```

# 2026.04.28

1. Analyze the recent blades data saving feature;
2. Implement ESCalibratingVoltage;

# 2026.04.29

1. Implement the TableNavigator class to navigate the table;
2. Analyze the transformer failed issues.

# 2026.04.30

1. Complete the ESCalibratingVoltageWidget;
2. Start to implement the impedance calibrating function.

# 2026.05.07

1. Implement ESCalibratingImpedanceWidget;
2. Test the open loop error on Beijing server;

#  2026.05.08

1. Prepare Chinese text table to the market staff;
2. Adding cutted voice feather.

Integrated Mainframe legacy issues:

1. The ligasure side should use "ElectroSurgery" in multi language.
2. Calibrating summary widget.
3. The two sides should both have voice control.

#  2026.05.09

1. Implement ESCalibratingSummaryWidget;
2. Split calibrating data from ic11 source code.

#  2026.05.10

#  2026.05.11

```c

float calculate_impedance(uint16_t power, uint16_t current) {
	float fAngel = 0.0;
	float fPhaseDiff = 0.0;
	float fRes = 0.0;
	float fPower = 0.0;
	float fCurrent = 0.0;

	const float fangel_90 = 1.46;
	const float fangel_0  = 2.88;
	const float currentcoff = 2.97;
	const float powercoff = 0.02887;

	fPhaseDiff = 3.0*phase/4095;
	fAngel = ((0 - 90.0)/(fangel_0 - fangel_90)*(fPhaseDiff - fangel_90)) + 90;

	if(fabs(cos(fAngel*3.1415926/180)) < 1e-6)
		return;

	if(fabs(fCurrent) < 1e-6)
		return;

	fPower = 3.0*power/4095.0;
	fCurrent = 3.0*current/4095.0;

	fPower = fPower/powercoff;
	fCurrent = fCurrent/currentcoff;

	fRes = (fPower)/cos(fAngel*3.1415926/180);
	fRes = fRes/(fCurrent*fCurrent);

	return fRes;
}

```

# 2026.05.14

run1: 3799 + 1500 = 5300 short
run1:
run4: 2495 + 400 = 2895  short

```c
const float Current_Value [2][15] =
{
	{409, 420, 449, 466, 514, 585,   622,  667,  750,  804, 885, 1001,  1211, 1542, 1960},//内部计数值；
	{57,  76,  113, 139, 217, 355,   424 , 514,  678,  783, 940,  1198,  1610, 2281, 3096} //实际电流值
};
```

run4: 2281 + 400 = 2681

# Shenzhen Number 1

## Short circuit (under run4 state)

impedance : 15
current :   current+value[1][13] + 1250
pid_number: 60

## Open circuit

380 --> 460 --> 600 --> 800, make sure the app can run to run4 with out printf.

run1: open_current_num > 380 确保程序跑到run4
run4: open_circuit_pid_num_limit = 57

other: 62
| Number | open circuit pid num |
| --- | --- |
| 1_plus | 57 |
| sh2603105006008 | 62 |

# 2026.05.19

1. Implement calibrating timeout mechanism;

## Task list

1. calibrating timeout mechanism; Done.
2. Disabled ligasure voices.
3. Optimizing the calibrating ui;
4. calibrating flashing function;
5. Audio asset


# 2026.05.20

1. Adding electrosurgery software version function;


# 2026.05.24

ZA Bank（众安银行）、天星银行、livi Bank

汇丰，再众安，最后中银

# 2026.05.25

| Time slot | first half | seconf half |
| --- | --- | --- |
|9:00 - 10:00|investgate hk bank issue| Nothing |
|10:00 - 11:00|investgate tencent video get | google test 60% |
|11:00 - 12:00| Google test stuffs |

1. Unittest cases set up;
2. Translate Text for mainframe;

# 2026.05.27

文天祥传 110 page

# 2026.05.28

| Time slot | first half | seconf half |
| --- | --- | --- |
|9:00 - 10:00|investgate hk bank issue| Nothing |
|10:00 - 11:00|investgate tencent video get | google test 60% |
|11:00 - 12:00| Google test stuffs |

1. Setting up googletest for ultrasurgery;
2. Write the audiotest for ultrasurgery;
3. Try to add new audio files to integrated mainframe.

# 2026.05.29

1. Denugging new audio playing issues;
2. Complete the mainframe's new audio playing feature.

## side project

StartAutoFollow --> Start to cut

EndAutoFollow --> finish to


# 2026.06.01

1. Start to implement the qt layer of flashing calibrating data;
2. Implement the audio chip verify cases;

# 2026.06.02

1. Continue with the audio chip verify cases;
2. Continue with the qt layer of flashing calibrating data;

# 2026.06.05

1. Start to implement the ic10 layer of flashing calibrating data;
2. ligasure ui coordination;
3. Trouble shooting for the new audio chip issue.

# 2026.06.08 - 2026.06.12

1. Start to design the architecture of calibration manager;
2. Complete the design the architecture of calibration manager;
3. implement the qt layer of the calibration manager;
4. implement the ic10 layer of the calibration manager;
5. implement the ic11 layer of the calibration manager;

# 2026.06.15

1. The calibration flashing architecture design;

# 2026.06.16

1. Debugging the ic11 layer of calibration manager;
2. FPGA developments refreshing;

Transfer to Wen Dao: (10000 - 38)*4 = 39848
Wen Dao Back:  16000 + 15962

# 2026.06.24

Go to advtech to solve the core board issues;

## Yanhua trip

| Board | issues |
| --- | --- |
|new processed| 开机log ok, 启动后字符不显示|
|old bad|  |

## 潮汕话

ti 桃  --> 去玩
阿弥陀佛 --> a mi tang hu

# 2026.06.25

```bash
./ultrasurgery_tests --gtest_filter=AudioTest.chip_verify --gtest_repeat=50 2>&1 | tee audio_verify.log
```

1. Audio chip verifying;
2. Flashing function implement, including flash ACK/NACK

## New Ultrasurgery software Requirements


发送给微控制器

| File | Time |
| --- | --- |
|001startup.mp3| 3s |
|002startup_bubble.mp3| 1s |
|003max.mp3 | 10s |
|004min.mp3 | 19s |
|005kn_test.mp3 | 2s |
|006ready.mp3 | 2s |
|007poweroff.mp3 | 2s |
|008poweroff_ding.mp3 | 2s|
|009warning.mp3 | 2s |
|010tip.mp3 | 1s |
|011lick.wav | Less than 1s|
|012finished.mp3 |Less than 1s|
|013cut.mp3 | 9s |
|014_es_working.mp3 | 9s |
|015_es_ending.mp3 | Less than 1s |
|016_es_warning.mp3 | 几乎一秒 |

# 2026.07.01

1. Flashing system;
2. Audio check;

# 2026.07.02


eb 90 85 09 50 57 45 53 30 30 30 31 00 44 36

# 2026.07.12

## 鲲鹏径第三段

起点：大井山警务工作室
终点：深圳市长龙书院

## 鲲鹏径第四段

起点：大井山警务工作室
终点：深圳市长龙书院

# 2026.07.13

## prompt for kernel

请您扫描整个目录

source目录下有uboot和linux kernel

# 2026.07.16

## one wired dev

 source\linux-3.2.0-psp04.06.00.11\drivers\w1\slaves\w1_gxe00.c

 ```
[    1.195565] Set the master secret in the Software SHA-256
[    1.201222]  w1_ds28e25_read_status(
[    1.224787] result : 0
[    1.227257] Read-Authenticate with unique secret
[    1.412118] result : 0
[    1.414588] DS28E25 Setup Example: SUCCESS
[    1.418879] --------------------------------------------------
[    1.425002] w1_ds28e25_setup_device success
[    1.632093] w1_ds28e25_verifymac result : 0
[    1.636477] w1_ds28e25_add_slave end, skip_setup=1, err=0
 ```

# 2026.07.20

```bash
git submodule add https://codeup.aliyun.com/657ffdee7f217c9ca94d71c8/purwell/ultrasurgery source
```

# 2026.07.27

| Desc | Contact | pri |
| --- | --- | --- |
| 后入易高潮少妇 | 1214178295，920198374 |
| 主动丰满型小妞儿| 1844681938 | 600 |
| 大奶长腿极骚妹子 |wang1237690 zz113118cy 2433193982 | 800 |
| 白皙结实的嫩妹 | 599801540 | 800 |


| ppm | name |
| --- | --- |
| logo_linux_clut224_black.ppm | black |
| logo_linux_clut224_purwell.ppm | purwell |
| logo_linux_clut224_mesic.ppm | mesic |
| logo_linux_clut224_lepuyiliao-poweron.ppm | lepu |
| | ledong |
| | lcm |
| | bcm |

Procision 1000 超声能量平台基于TI的ARM Cortex-A8处理器与Intel的cyclone IV FPGA的双芯配置，兼具强大的智能计算能力，和高速实时数据采集，以及精准的基于AI的反馈能量控制。软件界面灵活多变，自由度高，国际化轻松定制，提供了完美的扩展性与丝滑的用户体验。


# 2026.08.10

| Number | Language |
| --- | --- |
| 1 | 中文 |
| 2 | 德语 |
| 3 | 英语 |
| 4 | 西班牙语 |
| 5 | 法语 |
| 6 |  意大利语 |
| 7 | 葡萄牙语 |
| 8 | 俄罗斯语 |




send cmd: "7e 03 0e 0d ef" | stop
send cmd: "7e 05 41 00 08 4c ef" | play 0008 file
send cmd: "7e 04 33 02 35 ef   | loop mode


01-00:02:21.468 [Warning]:[ActualAudio::run:74]  AudioThread send cmd: "7e 03 0e 0d ef"
01-00:02:21.529 [Warning]:[ActualAudio::run:74]  AudioThread send cmd: "7e 05 41 00 08 4c ef"
01-00:02:21.590 [Warning]:[ActualAudio::run:74]  AudioThread send cmd: "7e 04 33 02 35 ef"
01-00:02:22.467 [Critical]:[TraceFreq::setParam:677]  set current: 420
01-00:02:22.551 [Critical]:[TraceFreq::setParam:677]  set current: 400
01-00:02:22.635 [Critical]:[TraceFreq::setParam:677]  set current: 380
01-00:02:23.274 [Warning]:[TraceFreq::run:553]  time:5820 frequency:55195.54,Iprms:0.28,Vprms:23.59,phase:-21.73, Ravg:85.33, Pavg:6.06, Rins:70.34
01-00:02:23.291 [Warning]:[TraceFreq::run:553]  time:5838 frequency:55197.00,Iprms:0.28,Vprms:21.91,phase:-32.08, Ravg:79.09, Pavg:5.14, Rins:65.19
01-00:02:23.300 [Warning]:[TraceFreq::run:553]  time:5846 frequency:55198.50,Iprms:0.28,Vprms:17.57,phase:-45.86, Ravg:63.62, Pavg:3.38, Rins:52.44
01-00:02:23.307 [Warning]:[TraceFreq::run:553]  time:5855 frequency:55200.87,Iprms:0.28,Vprms:15.46,phase:-52.72, Ravg:55.80, Pavg:2.59, Rins:45.99
01-00:02:23.316 [Warning]:[TraceFreq::run:553]  time:5863 frequency:55203.67,Iprms:0.28,Vprms:14.91,phase:-49.65, Ravg:53.98, Pavg:2.67, Rins:44.50
01-00:02:23.324 [Warning]:[TraceFreq::run:553]  time:5872 frequency:55206.27,Iprms:0.28,Vprms:14.44,phase:-40.72, Ravg:52.15, Pavg:3.03, Rins:42.99
01-00:02:23.341 [Warning]:[TraceFreq::run:553]  time:5889 frequency:55208.02,Iprms:0.28,Vprms:14.33,phase:-25.29, Ravg:51.60, Pavg:3.60, Rins:42.53
01-00:02:23.358 [Warning]:[TraceFreq::run:553]  time:5905 frequency:55208.03,Iprms:0.28,Vprms:14.11,phase:-36.64, Ravg:50.85, Pavg:3.14, Rins:41.92
01-00:02:23.361 [Warning]:[KeyInput::onKeyEvent:258]  isPressed : true , press : false
01-00:02:23.362 [Warning]:[KeyInput::onKeyEvent:275]  "button16 release"
01-00:02:23.363 [Warning]:[MainController::changeStatus:139]  status changed from ActivateMax to Idle
01-00:02:23.366 [Warning]:[ActualAudio::run:74]  AudioThread send cmd: "7e 03 0e 0d ef"
01-00:02:23.384 [Warning]:[TraceFreq::run:616]  Resistance interval = [41.9154, 276.31]
01-00:02:23.386 [Warning]:[TraceFreq::run:617]  last_activate_time =125783
01-00:02:23.386 [Warning]:[TraceFreq::run:618]  cutting consumed=5905 ms
01-00:02:23.387 [Critical]:[TraceFreq::setParam:677]  set current: 355
01-00:02:23.388 [Critical]:[TraceFreq::setParam:677]  set current: 330
01-00:02:23.389 [Critical]:[TraceFreq::setParam:677]  set current: 305
01-00:02:23.390 [Critical]:[TraceFreq::setParam:677]  set current: 280
01-00:02:23.391 [Critical]:[TraceFreq::setParam:677]  set current: 255
01-00:02:23.392 [Critical]:[TraceFreq::setParam:677]  set current: 230
01-00:02:23.393 [Critical]:[TraceFreq::setParam:677]  set current: 205
01-00:02:23.394 [Critical]:[TraceFreq::setParam:677]  set current: 180
01-00:02:23.395 [Critical]:[TraceFreq::setParam:677]  set current: 155
01-00:02:23.396 [Critical]:[TraceFreq::setParam:677]  set current: 130
01-00:02:23.397 [Critical]:[TraceFreq::setParam:677]  set current: 105
01-00:02:23.399 [Critical]:[TraceFreq::setParam:677]  set current: 80
01-00:02:23.404 [Critical]:[TraceFreq::setParam:677]  set current: 55
01-00:02:23.405 [Critical]:[TraceFreq::setParam:677]  set current: 30
01-00:02:23.406 [Critical]:[TraceFreq::setParam:677]  set current: 5
01-00:02:23.407 [Critical]:[TraceFreq::setParam:677]  set current: 1
01-00:02:23.427 [Warning]:[ActualAudio::run:74]  AudioThread send cmd: "7e 03 0e 0d ef"
01-00:02:23.709 [Warning]:[ActualAudio::run:74]  AudioThread send cmd: "7e 03 0e 0d ef"
01-00:02:26.561 [Warning]:[MainController::changeStatus:139]  status changed from Idle to HandleOff
01-00:02:26.594 [Warning]:[Container::push:186]  push widget:  HandleOffWidget(0x35dea8, name = "HandleOffWidget")
01-00:02:26.595 [Warning]:[ActualAudio::run:74]  AudioThread send cmd: "7e 05 41 00 0a 4e ef"
01-00:02:26.656 [Warning]:[ActualAudio::run:74]  AudioThread send cmd: "7e 04 33 04 33 ef"