# mavlink

## 介绍

## 功能说明

### 基本功能

#### 实例创建

根据参数可创建多个Mavlink实例

可支持串口通信，包括DMA

可支持UDP通信

可给各实例分配通道channel

#### 数据发送

mavlink库接口与调用

可订阅其他模块的ORB并转成对应message

可灵活定义要发送的message以及频率

根据通信质量和发送的数据量控制通信速率

#### 数据接收

mavlink库接口与调用

可接收并解析message并转成对应的ORB发布

#### 转发forward

各实例之间可实现message互相转发

#### 帮助命令

可启动/停止

可查看当前各实例运行状态

设置远端IP地址并写入参数

### 微服务

#### 参数服务parameter

#### 任务规划服务mission

#### ulog

#### shell

#### ftp

#### 时间同步



- 接收消息函数的注册：

MAVLINK_HANDLE_DEFINE，处理接受处理mavlink消息

- 发送消息函数注册：

MAVLINK_STREAM_DEFINE，定周期发送mavlink消息





## 开发



### 虚拟串口

fcu/board/cubmx/Core/Src/stm32f7xx_hal_msp.c

```c
void HAL_PCD_MspInit(PCD_HandleTypeDef* pcdHandle){
    
}
```



rt-thread/components/drivers/usb/class/cdc_vcom.c





## MAVLink协议

![image-20230309141927403](imgs\image-20230309141927403.png)



### 消息长度

以HEARTBEAT为例，其长度包括：

|起始位|帧头长度|payload长度|校验长度|签名长度|=1+9+payload+2+13

例如代码中：

```c
MAVLINK_START_UART_SEND(chan, header_len + 3 + (uint16_t)length + (uint16_t)signature_len);
```



校验：

CRC16_MCRF4XX
