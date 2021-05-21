[我的个人主页](https://xbchub.github.io/)

# VHDLCourseDesign
基于FPGA的最小值次小值不同算法求解、不同排序算法、使用OLED显示屏输出结果

## 报告

![初始化.jpg](https://i.loli.net/2021/05/21/6uLAq4BCWoKaGMU.jpg)
![目录.png](https://i.loli.net/2021/05/21/MraoGhcQ3TjD1Kx.png)

## Usage
本项目基于Xilinx ZedBoard，使用vivado 2018.3进行开发，仅使用了ZedBoard的PL部分，即纯FPGA实现，且未对内核ip进行修改。

代码位于目录 `./VHDLCourseDesign.srcs/` 

其中`./VHDLCourseDesign.srcs/sources_1/new`为设计，`./VHDLCourseDesign.srcs/constrs_1/new`为约束，`./VHDLCourseDesign.srcs/sim_1/new`为仿真

controller.vhd为顶层文件，默认采用最小次小的方案A与排序的方案A

## 简介
建立数据处理器-控制器模型，如图2-1所示。将数据信息和控制信息输入到控制器，然后经由控制器传送到数据处理器中进行处理，得到最小次小值以及排序结果，最后通过ASCII译码以及SPI协议传输到OLED屏幕和LED信号灯中进行实时显示。

将整体分为如下5个子模块：最小次小值查找、排序、信息输入、OLED显示、控制器。

### 不同方案选择
最小次小值查找和排序这两个模块体现了我们的算法，因此我们提出了不同方案，权衡时间与空间，力求减少资源占用，体现FPGA并行计算的优势。对于最小次小值查找，我们提出了4种方案：冒泡法并行比较、两两并行比较及其两种改进算法；对于排序，我们提出了两种方案：并行冒泡排序、串行比较排序。

### 最小次小比较
| 方案	| Slice LUTs	| Flip Flop	| Latch	| 计算速度 |
| --    | --         | --         | --    | --      |
|方案a	|130	|0	|0	|组合逻辑延时|
|方案b	|120	|0	|0	|组合逻辑延时|
|方案c	|114	|0	|0	|组合逻辑延时|
|方案d	|73	|33	|16	|2个时钟周期|

### 排序方案比较
| 方案	| Slice LUTs	| Flip Flop	| Latch	| 计算速度 |
| --    | --         | --         | --    | --      |
|方案a	|278	|0	|0	|组合逻辑延时|
|方案b	|143	|11	|0	|7个时钟周期|

### OLED原理
ZedBoard板载了OLED模块，其型号为UG-2832HSWEG04。为实现OLED显示，总体需分为两个初始化与显示两个模块，这两个模块中还包含了三个小模块，分别为通过SPI协议控制器发送数据、延时、ASCII码查表。
该OLED显示屏由128×32点阵构成，每8×8点阵表示一个字符，即一共可以显示16×4个字符，每8行点阵或每1行字符称为1页。

|引脚名|	功能|
|--|--|
|VDD	|逻辑门的电源供应|
|VBAT	|DC/DC转换电路的电源供应|
|RES#	|控制器和驱动的复位端，低电平有效|
|D/C#	|数据/命令控制，高电平传输数据，低电平发送命令|
|SCLK	|时钟序列|
|SDIN	|输入数据序列，在SCLK上升沿采集|

在进行OLED显示前，需要通过发送若干命令将OLED模块初始化。为此，采用状态机完成设计，该状态机分为外层与内层，每执行若干个外层状态，便会进入内层状态执行发送数据或延时。为方便进行设计，通过after_state表示由内层状态执行结束要跳转的下一个外层状态。

初始化状态转换图：

![image](https://user-images.githubusercontent.com/60500670/110566392-eb6a1500-818a-11eb-89da-818b52cca95d.png)


OLED显示模块根据获取的输入数据、最小值次小值结果或排序结果以及当前的状态，生成不同的屏幕内容并发送给OLED显示。为方便获取数据并转为对应的ASCII码值、将生成的ASCII码放置于屏幕相应的位置，将这一过程封装为一个函数供调用；为实现更新屏幕这一功能，需要逐页发送命令、逐ASCII码发送数据，故采用状态机进行设计。该状态机分为外层、中间层与内层。外层为整体的控制，包括准备、调用函数得到待更新的屏幕矩阵、等待。中间层有两个步骤，分别为发送ASCII码的每一位查表数据、发送命令更新屏幕。每更新一个字符，均要经过SendChar0 ~ 7查表发送，发送完一页后复位D/C，发送命令指向下一页，再置位D/C继续发送数据。内层状态大致与OLED_init相同，为使用SPI控制器发送具体数据或命令给OLED、以及延时。
当输入数据每4位小于10时，将其与30H相加即为其ASCII码，当输入数据每4位大于等于10时，将其与40H相加再减去9才为其ASCII码。构造每个元素为8位位矢量的16×4矩阵，则恰好与屏幕相对应，再根据矩阵元素的下标，替换进矩阵的指定位置，即完成了屏幕数据的储存。经验证，二维矩阵与函数在vivado 2018.3与ZedBoard环境下可综合。

显示状态转换图：

![image](https://user-images.githubusercontent.com/60500670/110566419-f91f9a80-818a-11eb-8084-52c9dc6d3f6c.png)


### 资源占用
![image](https://user-images.githubusercontent.com/60500670/110566260-beb5fd80-818a-11eb-8ccb-8765b80048bf.png)

## OLED程序参考
https://github.com/mmattioli/ZedBoard-OLED


