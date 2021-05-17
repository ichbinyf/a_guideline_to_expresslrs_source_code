ExpressLRS代码通读

0.前言
近来，以高刷和低延迟为目标的ExpressLRS大热，GitHub代码已经更新到1.0RC5，开源硬件和商业成品硬件已经成熟。Discord社区和相关社交群人气较高。
为更好了解该协议，也方便各位有能力做二次开发的模友加入其中，我撰写此文。其中如有错漏欢迎指出。

1.开发环境
Platform.io是ExpressLRS的IDE（集成开发环境）。
首先需要安装Visual Studio Code，然后在插件中心找到并安装Platform.io。全过程可以参考该篇文章：https://blog.csdn.net/happyjoey217/article/details/113177118
国内开发者遇到的第一个问题是Platform.io的依赖装不上，在此推荐一个方法，VSS菜单->File->Preferences->Settings，检索proxy，在Http:Proxy中设置代理地址。

2.导入项目
下载源码时，打开https://github.com/ExpressLRS/ExpressLRS/tags，找到对应版本的源码并下载解压到本地。
VSS中打开Platform.io，选择源码文件夹并打开。

3.编译和下载到硬件。
VSS->Platform.io（VSS左侧小图标的“蚂蚁头”）->左侧PROJECT TASKS选择对应的硬件类型，点击展开（第一次点击时，platform.io会自动下载和准备相关环境，耗时较久）。
进入VSS的文件界面（VSS左侧小图标的“文件”），打开user_defines.txt文件，填上所需的定义。
例1：diy 2400 rx
```-DMY_BINDING_PHRASE="myelrs2g4"
-DRegulatory_Domain_ISM_2400
-DHYBRID_SWITCHES_8
-DTLM_REPORT_INTERVAL_MS=320LU
-DFEATURE_OPENTX_SYNC
-DUART_INVERTED
-DLOCK_ON_FIRST_CONNECTION```

例2：diy 2400 tx
```-DMY_BINDING_PHRASE="myelrs2g4"
-DRegulatory_Domain_ISM_2400
-DHYBRID_SWITCHES_8
-DTLM_REPORT_INTERVAL_MS=320LU
-DFEATURE_OPENTX_SYNC
-DUART_INVERTED```

点击build编译代码，插上对应的硬件（串口或者STlink），点击Upload即可上传到硬件。

4.代码结构
4.1硬件定义
targets目录下是各个硬件的配置文件，每个ini文件对应一类硬件，ini中的每个env对应一个具体硬件，每个具体硬件描述了三项内容extents所需环境，build_flags编译时的预定义，src_filter对应的源代码。

例3：diy_900.ini文件的一个配置项
```[env:DIY_900_TX_ESP32_SX127x_E19_via_UART]
extends = env_common_esp32
build_flags =
	${env_common_esp32.build_flags}
	${common_env_data.build_flags_tx}
	-D TARGET_EXPRESSLRS_PCB_TX_V3
	-D TARGET_1000mW_MODULE
src_filter = ${env_common_esp32.src_filter} -<rx_*.cpp>```

该配置文件包括所有915M diy硬件类型。该配置项为“esp32+Ebyte E19射频模块”硬件。
首先env_common_esp32是该硬件的代码环境，该环境可在common.ini中找到更具体的信息。
其次build_flags包括了4个预定义，env_common_esp32.build_flags可以在common.ini中找到（	-D PLATFORM_ESP8266=1，	-D VTABLES_IN_FLASH=1），common_env_data.build_flags_tx可以在common.ini中找到，第3和第4个预定义（-D TARGET_EXPRESSLRS_PCB_TX_V3，-D TARGET_1000mW_MODULE）。
预定义的作用相当于define PLATFORM_ESP8266=1，则编译程序时编译器自动选择对应的代码。对CPP语法不熟悉的小伙伴可以参考https://www.cnblogs.com/zi-xing/p/4550246.html。
最后src_filter表示，该硬件所需的源码不包括“rx_”开头的“.cpp”文件。该硬件为高频头，英文常用tx表示（全称transmit），而rx常用来表示接收机（全称receive）。

4.2主流程代码
建议首先查看tx_main.cpp文件，该文件中的setup和loop函数。
有了解过Arduino的小伙伴应该熟悉这2个函数。Arduino程序下载到硬件后，硬件首先会运行setup函数，该函数运行完成后循环执行loop函数。相当于
```void main(){
    setup();
    while(1){
        loop();
    }
}```
setup函数和loop函数作用大致如下：
setup：(1)初始化串口(2)初始化LED和蜂鸣器、按钮(3)根据种子生成随机数初始化跳频的频率表（FHSSrandomiseFHSSsequence）(4)定义回调函数，其中最重要的是hwTimer.callbackTock=&timerCallbackNormal，硬件时钟回调timerCallbackNormal函数，函数会发送Lora数据包(5)初始化外设，包括硬件时钟、crsf（和opentx控交互数据）
loop：(1)更新LED灯(2)接受opentx控Lua脚本的设置(3)更新连接状态（连接或断开）(4)处理opentx控通过crsf传来的数据
大概流程就这样。关键还是整个主流程，然后大家可以找到对应的代码，做想做的事。

4.3高频头与控的交互代码
lib/CRSF以及lib/CrsfProtocol

4.4射频部分代码
lib/SX127xDriver
lib/SX1280Driver

4.5引脚定义
/include/targets.h可以根据硬件的预定义看到各个引脚的含义
例：
```#elif defined(TARGET_EXPRESSLRS_PCB_TX_V3)
#define GPIO_PIN_NSS 5
#define GPIO_PIN_DIO0 26
#define GPIO_PIN_DIO1 25
#define GPIO_PIN_MOSI 23
#define GPIO_PIN_MISO 19
#define GPIO_PIN_SCK 18
#define GPIO_PIN_RST 14
#define GPIO_PIN_RX_ENABLE 13
#define GPIO_PIN_TX_ENABLE 12
#define GPIO_PIN_RCSIGNAL_RX 2
#define GPIO_PIN_RCSIGNAL_TX 2 // so we don't have to solder the extra resistor, we switch rx/tx using gpio mux
#define GPIO_PIN_LED 27```

因为DIY_900_TX_ESP32_SX127x_E19_via_UART硬件的build_flag中有一个TARGET_EXPRESSLRS_PCB_TX_V3，和上述代码对应的上。
spi的片选信号是esp32的io5，E19射频模块的dio0对应esp32的io26，E19射频模块的dio1对应esp32的io25，spi的MOSI信号是esp32的io23，spi的MISO信号是esp32的io19，spi的时钟信号是esp32的io18。
E19射频模块的txen对应esp32的io13，E19射频模块的rxen对应esp32的io12.
然后比较有意思的是#define GPIO_PIN_RCSIGNAL_RX 2和#define GPIO_PIN_RCSIGNAL_TX 2，表示esp32的io2用作sport，和opentx控连接的sport。

5.最后
大概就这些，受限于篇幅，这里并不能带你弄懂所有Expresslrs的细节，做自己想做的事（tx的PPM输入、rx的PWM输出等等），只是跟大家梳理一下项目内容。
前置知识不多的，主要是Arduino（能点灯，抄着改个中断的程序就OK）。
最后的最后platform.io环境里有些常用的小知识点，Ctrl+鼠标左键可以进入函数的定义代码，右键点击函数可以跳转到定义或者引用的代码。
