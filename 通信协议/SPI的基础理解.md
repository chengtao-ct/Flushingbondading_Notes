**SPI是一种高速，全双工，同步的通讯协议（也是最傻的哪个），传输数据，本质是数据交换**


**物理层**
![](../assest/{1C1B5452-84E5-486B-BE5E-2E3BFF588DE1}.png)
![](../assest/{1362F5ED-6C92-4E77-97D6-57406978DEBA}.png)
1. CS：一机多从的特点，其中CS片选引脚，是来选择从机的
2. MOSI MISO:同时输入和输出的特点，即数据如同在俩个移位寄存器里面交换，说明读一定要写的特点，由MOSI(输出)，MISO(输入)控制
3. SCK：是同步通信，意味着有时钟线(SCK)控制
4. 移位寄存器： 配合SCK将并行的数据改变成串行输入到MOSI,反之亦然
5. 传输速度：取决于时钟速率
---

所以总的来说SPI有四根线，这种高硬件化的确提供了高速，双向同行的优势，但成本也上去了

---

**协议层**
![](../assest/{38B88709-AFA8-40BC-88B0-E1D7C143780F}.png)
1. 四种模式的选择：由CPHA,CPOL俩者共同决定
![](../assest/{2099BDDE-9341-4B59-AA2E-3E39E5E6205E}.png)
2. CPHA的作用：SPI通讯设备处于空闲状态时。
3. CPOL的作用：SPI采样时刻，边沿采样点

---

**采样和移位的理解**
根据SCK的时钟信号和CPOL的选择，在电压变换的时候（上升或者下降），一部分是从移位寄存器里面采样到MOSI通道，另一部分是进行移位寄存器电平反转，来表示数据。

---

**在stm32里面的基础设置**
![](../assest/{355B7A3F-8CE5-4F57-A44F-E8329DA71638}.png)

*主要涉及到用位控制寄存器去硬件化SPI，详细见 > STM32F4xx中文手册*

涉及到一些寄存器的配置：
1. SPI_DR (Data Register，数据寄存器)：主要用于缓冲的接发作用，发送和接收
2. SPI_SR (Data Status，状态寄存器)：检测TXE(Transmit Buffer Empty)，RXNE(Receive Buffer Not Empty)
3. SPI_CR (Control Register 控制寄存器)：配置SCK,CPOL,CPHA,MSTR
![](../assest/{ED607FFA-42C5-4537-A394-FDB3F17E5D0D}.png)

**TIPS和注意事项**

来源于Genimi
![](../assest/{53F105B2-B770-4CEC-9773-F3CE0E2708B3}.png)

**出现的问题**

*主要有内存问题，硬件问题，芯片底层问题*

1. SPI_Timeout引起的俩次中断导致的双重释放，导致死机
2. 触摸屏没有反应->内存不足问题
3. 在高速的情况下，时钟发生错位，要用tick_delay调整采样点
![](../assest/{3DC90FE2-D7D7-4F76-9060-45AC35245F17}.png)
4. 回环测试来检测是主机有传输问题还是从机问题，利用了全双工的性质
[B站的一个视频](https://www.bilibili.com/video/BV1GwijBXEs9/?spm_id_from=333.1387.homepage.video_card.click&vd_source=603f3c284e76fbe772654083937e3fac)