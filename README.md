# ADC/UART data capturing using xWR1843/AWR2243 with DCA1000

* for xWR1843: capture both raw ADC IQ data and processed UART point cloud data simultaneously in Python and C(pybind11) without mmwaveStudio
* for AWR2243: capture raw ADC IQ data in Python and C(pybind11) without mmwaveStudio


## Introduction

* 该模块主要分为两部分，mmwave和fpga_udp:
  * mmwave修改自[OpenRadar](https://github.com/PreSenseRadar/OpenRadar)，用于配置文件读取、串口数据发送与接收、原始数据解析等。
  * fpga_udp修改自[pybind11 example](https://github.com/pybind/python_example)以及[mmWave-DFP-2G](https://www.ti.com/tool/MMWAVE-DFP)，用于通过C语言编写的socket代码从网口接收DCA1000发回的高速的原始数据。对于AWR2243这种没有片上DSP及ARM核的型号，还实现了利用FTDI通过USB发送指令用SPI控制AWR2243的固件写入、参数配置等操作。
* TI的毫米波雷达主要分两类，只有射频前端的和自带片上ARM及DSP/HWA的。前者型号有[AWR1243](https://www.ti.com/product/AWR1243)、[AWR2243](https://www.ti.com/product/AWR2243)等，后者型号有[xWR1443](https://www.ti.com/product/IWR1443)、[xWR6443](https://www.ti.com/product/IWR6443)、[xWR1843](https://www.ti.com/product/IWR1843)、[xWR6843](https://www.ti.com/product/IWR6843)、[AWR2944](https://www.ti.com/product/AWR2944)等。
  * 对于只有射频前端的雷达传感器，一般是通过SPI/I2C接口向其发送控制及配置指令，并通过CSI2/LVDS接口输出原始数据。其中SPI接口可以用DCA1000板载的FTDI芯片转为USB协议直接用电脑控制，LVDS接口的数据也能用DCA1000板载的FPGA采集并转为UDP包通过以太网传输。本仓库实现了上述所有操作。
  * 对于自带片上ARM及DSP的雷达传感器，则可以烧录控制程序，用片上ARM对雷达传感器进行配置，用片上DSP处理原始数据并得到点云等数据，通过UART传输。其中原始数据除了送入片上DSP，还能配置为通过LVDS接口输出，并利用DCA1000板载的FPGA采集。本仓库实现了上述所有操作。当然，自带片上ARM及DSP的雷达传感器也有SPI/I2C等接口，也可以利用该接口对雷达传感器进行配置，并通过DCA1000板载的FTDI将SPI转为USB用电脑控制。[mmwaveStudio](https://www.ti.com/tool/MMWAVE-STUDIO)就是这样做的，但本仓库暂未实现该方法，可参考[mmWave-DFP](https://www.ti.com/tool/MMWAVE-DFP)自行实现。

## Introduction (English)
This module is mainly divided into two parts: mmwave and fpga_udp.

* mmwave is adapted from OpenRadar and is responsible for configuration file parsing, UART data transmission/reception, raw data parsing, and related functions.

* fpga_udp is adapted from a pybind11 example and mmWave-DFP-2G. It uses C-written socket code to receive high-speed raw data from the DCA1000 over Ethernet.
For radar chips like AWR2243, which do not have on-chip DSP or ARM cores, it also implements USB-to-SPI command transmission via FTDI for firmware flashing, parameter configuration, and other SPI-based operations.

### TI mmWave Radar Categories
TI mmWave radar sensors are mainly divided into two categories:

* RF front-end only: such as AWR1243, AWR2243, etc. With built-in ARM, DSP/HWA: such as xWR1443, xWR6443, xWR1843, xWR6843, AWR2944, etc. For RF Front-End Only Radar Sensors
These sensors are usually controlled/configured via SPI/I2C, and output raw ADC data via CSI2/LVDS interfaces. The SPI interface can be converted to USB using the DCA1000's onboard FTDI chip, enabling control via a PC.
The LVDS data can be captured by the DCA1000's onboard FPGA, converted into UDP packets, and sent over Ethernet.
This repository implements all of the above operations.

* For Radar Sensors with On-Chip ARM and DSP

You can:

* Flash a control program,

Use the on-chip ARM core to configure the radar sensor, Use the on-chip DSP to process raw data into point clouds, Transmit data over UART.

In addition:

Raw data can be routed both to the on-chip DSP and out via LVDS, captured via the DCA1000 FPGA.

This repository also supports these functionalities.

Although these sensors also provide SPI/I2C interfaces and can technically be configured via SPI (using the DCA1000 FTDI for USB control), this method is not yet implemented in this repository.
However, it is supported by TI’s mmWaveStudio, and you can refer to mmWave-DFP to implement it yourself.

## Prerequisites

### Hardware
#### for xWR1843
* Connect the micro-USB port (UART) on the xWR1843 to your system
* Connect the xWR1843 to a 5V barrel jack
* Set power connector on the DCA1000 to RADAR_5V_IN
* boot in Functional Mode: SOP[2:0]=001
  * either place jumpers on pins marked as SOP0 or toggle SOP0 switches to ON, all others remain OFF
* Connect the RJ45 to your system
* Set a fixed IP to the local interface: 192.168.33.30
#### for AWR2243
* Connect the micro-USB port (FTDI) on the DCA1000 to your system
* Connect the AWR2243 to a 5V barrel jack
* Set power connector on the DCA1000 to RADAR_5V_IN
* Put the device in SOP0
  * Jumper on SOP0, all others disconnected
* Connect the RJ45 to your system
* Set a fixed IP to the local interface: 192.168.33.30

### Software
#### Windows
 - Microsoft Visual C++ 14.0 or greater is required.
   - Get it with "[Microsoft C++ Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/)"(Standalone MSVC compiler) or "[Visual Studio](https://visualstudio.microsoft.com/downloads/)"(IDE) and choose "Desktop development with C++"
 - FTDI D2XX driver and DLL is needed. 
   - Download version [2.12.36.4](https://www.ftdichip.com/Drivers/CDM/CDM%20v2.12.36.4%20WHQL%20Certified.zip) or newer from [official website](https://ftdichip.com/drivers/d2xx-drivers/).
   - Unzip it and install `.\ftdibus.inf` by right-clicking this file.
   - Copy `.\amd64\ftd2xx64.dll` to `C:\Windows\System32\` and rename it to `ftd2xx.dll`. For 32-bit system, just copy `.\i386\ftd2xx.dll` to that directory.
#### Linux
 - `sudo apt install python3-dev`
 - FTDI D2XX driver and .so lib is needed. Download version 1.4.27 or newer from [official website](https://ftdichip.com/drivers/d2xx-drivers/) based on your architecture, e.g. [X86](https://ftdichip.com/wp-content/uploads/2022/07/libftd2xx-x86_32-1.4.27.tgz), [X64](https://ftdichip.com/wp-content/uploads/2022/07/libftd2xx-x86_64-1.4.27.tgz), [armv7](https://ftdichip.com/wp-content/uploads/2022/07/libftd2xx-arm-v7-hf-1.4.27.tgz), [aarch64](https://ftdichip.com/wp-content/uploads/2022/07/libftd2xx-arm-v8-1.4.27.tgz), etc.
 - Then you'll need to install the library:
   - ```
     tar -xzvf libftd2xx-arm-v8-1.4.27.tgz
     cd release
     sudo cp ftd2xx.h /usr/local/include
     sudo cp WinTypes.h /usr/local/include
     cd build
     sudo cp libftd2xx.so.1.4.27 /usr/local/lib
     sudo chmod 0755 /usr/local/lib/libftd2xx.so.1.4.27
     sudo ln -sf /usr/local/lib/libftd2xx.so.1.4.27 /usr/local/lib/libftd2xx.so
     sudo ldconfig -v
     ```


## Installation

 - clone this repository
 - for Windows:
   - `python3 -m pip install --upgrade pip`
   - `python3 -m pip install --upgrade setuptools`
   - `python3 -m pip install ./fpga_udp`
 - for Linux:
   - `sudo python3 -m pip install --upgrade pip`
   - `sudo python3 -m pip install --upgrade setuptools`
   - `sudo python3 -m pip install ./fpga_udp`


## Instructions for Use

#### General
1.  先按照[Prerequisites](#prerequisites)搭建运行环境
2.  再按[Installation](#installation)安装库
3.  未提及的模块在运行时若报错请自行查询添加
#### For xWR1843
1.  按[mmwave SDK](https://www.ti.com/tool/MMWAVE-SDK)的说明烧录xwr18xx_mmw_demo程序
2.  用[mmWave_Demo_Visualizer](https://dev.ti.com/gallery/view/mmwave/mmWave_Demo_Visualizer/ver/3.6.0/)调整参数并保存cfg配置文件
3.  打开[captureAll.py](#captureallpy)按需求修改并填入cfg配置文件地址及端口号后运行并开始采集数据
4.  打开[testDecode.ipynb](#testdecodeipynb)或[testDecodeADCdata.mlx](#testdecodeadcdatamlx)解析刚才采集的数据
5.  对参数不满意可以继续用[mmWave_Demo_Visualizer](https://dev.ti.com/gallery/view/mmwave/mmWave_Demo_Visualizer/ver/3.6.0/)调整或用[testParam.ipynb](#testparamipynb)修改并检验参数的合理性
#### For AWR2243
1.  烧录固件补丁至外部flash（该操作只需要执行一次，重启不会丢固件）
  - for Windows: `python3 -c "import fpga_udp;fpga_udp.AWR2243_firmwareDownload()"`
  - for Linux: `sudo python3 -c "import fpga_udp;fpga_udp.AWR2243_firmwareDownload()"`
  - 看见"MSS Patch version [ 2. 2. 2. 0]"则烧录成功
2.  打开[captureADC_AWR2243.py](#captureadc_awr2243py)按需求修改并填入txt配置文件地址并开始采集数据
3.  打开[testDecode_AWR2243.ipynb](#testdecode_awr2243ipynb)解析刚才采集的数据
4.  对参数不满意可以用[testParam_AWR2243.ipynb](#testparam_awr2243ipynb)修改并检验参数的合理性


## Example

### ***captureAll.py***
同时采集原始ADC采样的IQ数据及片内DSP预处理好的点云等串口数据的示例代码（仅xWR1843）。
#### 1.采集原始数据的一般流程
 1. 重置雷达与DCA1000(reset_radar、reset_fpga)
 2. 通过UART初始化雷达并配置相应参数(TI、setFrameCfg)
 3. (optional)创建从串口接收片内DSP处理好的数据的进程(create_read_process)
 4. 通过网口udp发送配置fpga指令(config_fpga)
 5. 通过网口udp发送配置record数据包指令(config_record)
 6. (optional)启动串口接收进程（只进行缓存清零）(start_read_process)
 7. 通过网口udp发送开始采集指令(stream_start)
 8. 启动UDP数据包接收线程(fastRead_in_Cpp_async_start)
 9. 通过串口启动雷达（理论上通过FTDI(USB转SPI)也能控制，目前只在AWR2243上实现）(startSensor)
 10. 等待UDP数据包接收线程结束+解析出原始数据(fastRead_in_Cpp_async_wait)
 11. 保存原始数据到文件离线处理(tofile)
 12. (optional)通过网口udp发送停止采集指令(stream_stop)
 13. 通过串口关闭雷达(stopSensor) 或 通过网口发送重置雷达命令(reset_radar)
 14. (optional)停止接收串口数据(stop_read_process)
 15. (optional)解析从串口接收的点云等片内DSP处理好的数据(post_process_data_buf)
#### 2."*.cfg"毫米波雷达配置文件要求
 - Default profile in Visualizer disables the LVDS streaming.
 - To enable it, please export the chosen profile and set the appropriate enable bits.
 - adcbufCfg需如下设置，lvdsStreamCfg的第三个参数需设置为1，具体参见mmwave_sdk_user_guide.pdf
    - adcbufCfg -1 0 1 1 1
    - lvdsStreamCfg -1 0 1 0 
#### 3."cf.json"数据采集卡配置文件要求
 - 具体信息请查阅TI_DCA1000EVM_CLI_Software_UserGuide.pdf
 - lvds Mode:
    - LVDS mode specifies the lane config for LVDS. This field is valid only when dataTransferMode is "LVDSCapture".
    - The valid options are
    - • 1 (4lane)
    - • 2 (2lane)
 - packet delay:
    - In default conditions, Ethernet throughput varies up to 325 Mbps speed in a 25-µs Ethernet packet delay. 
    - The user can change the Ethernet packet delay from 5 µs to 500 µs to achieve different throughputs.
       - "packetDelay_us":  5 (us)   ~   706 (Mbps)
       - "packetDelay_us": 10 (us)   ~   545 (Mbps)
       - "packetDelay_us": 25 (us)   ~   325 (Mbps)
       - "packetDelay_us": 50 (us)   ~   193 (Mbps)

### ***captureADC_AWR2243.py***
采集原始ADC采样的IQ数据的示例代码（仅AWR2243）。
#### 1.AWR2243采集原始数据的一般流程
 1. 重置雷达与DCA1000(reset_radar、reset_fpga)
 2. 通过SPI初始化雷达并配置相应参数(AWR2243_init、AWR2243_setFrameCfg)(linux下需要root权限)
 3. 通过网口udp发送配置fpga指令(config_fpga)
 4. 通过网口udp发送配置record数据包指令(config_record)
 5. 通过网口udp发送开始采集指令(stream_start)
 6. 启动UDP数据包接收线程(fastRead_in_Cpp_async_start)
 7. 通过SPI启动雷达(AWR2243_sensorStart)
 8. 1. (optional, 若numFrame==0则必须有)通过SPI停止雷达(AWR2243_sensorStop)
    2. (optional, 若numFrame==0则不能有)等待雷达采集结束(AWR2243_waitSensorStop)
 9. (optional, 若numFrame==0则必须有)通过网口udp发送停止采集指令(stream_stop)
 10. 等待UDP数据包接收线程结束+解析出原始数据(fastRead_in_Cpp_async_wait)
 11. 保存原始数据到文件离线处理(tofile)
 12. 通过SPI关闭雷达电源与配置文件(AWR2243_poweroff)
#### 2."mmwaveconfig.txt"毫米波雷达配置文件要求
 - TBD
#### 3."cf.json"数据采集卡配置文件要求
 - 同上

### ***realTimeProc.py***
实时循环采集原始ADC采样的IQ数据并在线处理的示例代码（仅xWR1843）。
#### 1.采集原始数据的一般流程
 1. 重置雷达与DCA1000(reset_radar、reset_fpga)
 2. 通过UART初始化雷达并配置相应参数(TI、setFrameCfg)
 3. 通过网口udp发送配置fpga指令(config_fpga)
 4. 通过网口udp发送配置record数据包指令(config_record)
 5. 通过网口udp发送开始采集指令(stream_start)
 6. 通过UART启动雷达（理论上通过FTDI(USB转SPI)也能控制，目前只在AWR2243上实现）(startSensor)
 7. UDP**循环**接收数据包+解析出原始数据+数据实时处理(fastRead_in_Cpp、postProc)
 8. 通过UART停止雷达(stopSensor)
 9. 通过网口udp发送停止采集指令(fastRead_in_Cpp_thread_stop、stream_stop)
#### 2."mmwaveconfig.txt"毫米波雷达配置文件要求
 - 略
#### 3."cf.json"数据采集卡配置文件要求
 - 略

### ***realTimeProc_AWR2243.py***
实时循环采集原始ADC采样的IQ数据并在线处理的示例代码（仅AWR2243）。
#### 1.AWR2243采集原始数据的一般流程
 - 略
#### 2."mmwaveconfig.txt"毫米波雷达配置文件要求
 - TBD
#### 3."cf.json"数据采集卡配置文件要求
 - 略

### ***testDecode.ipynb***
解析原始ADC采样数据及串口数据（仅IWR1843）的示例代码，需要用Jupyter(推荐VS Code安装Jupyter插件)打开。
#### 1.解析LVDS接收的ADC原始IQ数据
##### 利用numpy对LVDS接收的ADC原始IQ数据进行解析
 - 载入相关库
 - 设置对应参数
 - 载入保存的bin数据并解析
 - 绘制时域IQ波形
 - 计算Range-FFT
 - 计算Doppler-FFT
 - 计算Azimuth-FFT
##### 利用mmwave.dsp提供的函数对LVDS接收的ADC原始IQ数据进行解析
#### 2.解析UART接收的片内DSP处理过的点云、doppler等数据
 - 载入相关库
 - 载入保存的串口解析数据
 - 显示cfg文件设置的数据
 - 显示片内处理时间(可用来判断是否需要调整帧率)
 - 显示各天线温度
 - 显示数据包信息
 - 显示点云数据
 - 计算距离标签及多普勒速度标签
 - 显示range profile及noise floor profile
 - 显示多普勒图Doppler Bins
 - 显示方位角图Azimuth (Angle) Bins

### ***testDecode_AWR2243.ipynb***
解析原始ADC采样数据（仅AWR2243）的示例代码，需要用Jupyter(推荐VS Code安装Jupyter插件)打开。
#### 1.解析LVDS接收的ADC原始IQ数据
##### 利用numpy对LVDS接收的ADC原始IQ数据进行解析
 - 载入相关库
 - 设置对应参数
 - 载入保存的bin数据并解析
 - 绘制时域IQ波形
 - 计算Range-FFT
 - 计算Doppler-FFT
 - 计算Azimuth-FFT
 
### ***testParam.ipynb***
IWR1843毫米波雷达配置参数合理性校验，需要用Jupyter(推荐VS Code安装Jupyter插件)打开。
 - 主要校验毫米波雷达需要的cfg文件及DCA采集板需要的cf.json文件的参数配置是否正确。
 - 参数的约束条件来自于IWR1843自身的器件特性，具体请参考IWR1843数据手册、mmwave SDK用户手册、chirp编程手册。
 - 若参数满足约束条件将以青色输出调试信息，若不满足则以紫色或黄色输出。
 - 需要注意的是，本程序约束条件并非完全准确，故特殊情况下即使参数全都满足约束条件，也有概率无法正常运行。

### ***testParam_AWR2243.ipynb***
同上，AWR2243毫米波雷达配置参数的合理性校验。

### ***testDecodeADCdata.mlx***
解析原始ADC采样数据的MATLAB示例代码
 - 设置对应参数
 - 载入保存的bin原始ADC数据
 - 根据参数解析重构数据格式
 - 绘制时域IQ波形
 - 计算Range-FFT(一维FFT+静态杂波滤除)
 - 计算Doppler-FFT
 - 1D-CA-CFAR Detector on Range-FFT
 - 计算Azimuth-FFT

### ***testGtrack.py***
利用cppyy测试C语言写的gTrack算法。该算法为TI的群目标跟踪算法，输入点云，输出轨迹。

The algorithm is designed to track multiple targets, where each target is represented by a set of measurement points.
Each measurement point carries detection informations, for example, range, azimuth, elevation (for 3D option), and radial velocity.

Instead of tracking individual reflections, the algorithm predicts and updates the location and dispersion properties of the group.

The group is defined as the set of measurements (typically, few tens; sometimes few hundreds) associated with a real life target.

Algorithm supports tracking targets in two or three dimensional spaces as a build time option:
 - When built with 2D option, algorithm inputs range/azimuth/doppler information and tracks targets in 2D cartesian space.
 - When built with 3D option, algorithm inputs range/azimuth/elevation/doppler information and tracks targets in 3D cartesian space.
#### Input/output
 - Algorithm inputs the Point Cloud. For example, few hundreds of individual measurements (reflection points).
 - Algorithm outputs a Target List. For example, an array of few tens of target descriptors. Each descriptor carries a set of properties for a given target.
 - Algorithm optionally outputs a Target Index. If requested, this is an array of target IDs associated with each measurement.
#### Features
 - Algorithm uses extended Kalman Filter to model target motion in Cartesian coordinates.
 - Algorithm supports constant velocity and constant acceleartion models.
 - Algorithm uses 3D/4D Mahalanobis distances as gating function and Max-likelihood criterias as scoring function to associate points with an existing track.


## Software Architecture

### TBD.py
TBD
