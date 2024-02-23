# PX4代码移植

## vscode使用

### 快捷键

- ctrl+shift+\\：花括号起始位置和结束位置的跳转；
- alt+左箭头/右箭头：光标位置跳转；
- ctrl+alt+上箭头/下箭头：列光标；
- alt+o：源文件与头文件之间切换；

### 正则表达式

```shell
### 1. 
if (_cmd_sub.updated())
# 
if \(_.*.updated\(\)\)

### 2. 
param_int32_t _param_nav_dll_act{"NAV_DLL_ACT"};
#
param_[a-z]*[0-9]*_t (\w+)\{"(\w+)"\}
```



#### 示例一：参数替换

```shell
(ParamFloat<px4::params::CP_DIST>) _param_cp_dist
替换为
param_float_t _param_cp_dist{"CP_DIST"};
```

表达式为：

```shell
#
\(ParamFloat.+::(\w+)>\)(\w+),
#
param_float_t $2{"$1"};
```



#### 示例二

参数更新

```shell
param_int32_t _param_nav_force_vt{"NAV_FORCE_VT"};
替换为
_param_nav_force_vt.update();
```

表达式为：

```shell
param_int32_t (\w*)\{.*

$1.update();
```



#### 示例三

```c
param_find("AcceptRadius"),
替换为
param_t param_AcceptRadius = param_find("AcceptRadius");
```



表达式为：

```c
param_find\("(\w+)"\)
param_t param_$1 = param_find("$1");
```





```shell
// 
configure_stream_local\("DISTANCE_SENSOR"
// 替换为
//configure_stream_local\("DISTANCE_SENSOR"
                        
```





### 参数

#### 使用

定义句柄和变量

```

```

句柄初始化

```shell

```

更新参数

```shell
```





## 通用替换部分

### 延时

注意延时单位不一样

px4_usleep -> rt_thread_mdelay

### param定义

添加头文件

```c
#include "param.h"
```

每个参数定义后增加一个参数

```c
PARAM_DEFINE_INT32(SDLOG_MODE, 0, 0);
```



### uorb定义

### 整个文件夹替换

PX4_INFO -> LOG_I

PX4_WARN-> LOG_W

PX4_DEBUG -> LOG_D

PX4_ERR -> LOG_E

PX4_ERROR -> RT_ERROR

PX4_OK -> RT_EOK

PX4_ISFINITE -> isfinite

## Commander_cpp调试

校准

```shell
[101182] W/NO_TAG: Preflight Fail: no valid data from Compass 0
[101182] W/NO_TAG: Preflight Fail: no valid data from Compass 1
[101183] W/NO_TAG: Preflight Fail: no valid data from Compass 2
[101183] W/NO_TAG: Preflight Fail: no valid data from Compass 3
[101184] W/NO_TAG: Preflight Fail: no valid data from Baro 0
[101188] W/NO_TAG: system power unavailable
```

## mavlink

### 飞控固件版本

AUTOPILOT_VERSION ([ #148 ](http://mav-dev.cetcs.com/en/messages/common.html#AUTOPILOT_VERSION))

MAV_CMD_REQUEST_AUTOPILOT_CAPABILITIES

### 遇到的问题

#### lib库

为了临时编译，将lib库直接放到了apps文件夹下面。

#### 订阅数组

在PX4中，uORB::SubscriptionMultiArray定义了订阅数组，而nextpilot中没有，如何进行处理？？？



#### mission

在qemu仿真中，启动后，有时候会出现函数MavlinkMissionManager::init_offboard_mission()报错：

```shell
[1074] E/mavlink_mission: DM_KEY_MISSION_STATE lock failed
[1075] E/mavlink_mission: offboard mission init failed (-1)
```

- 出现该问题时，dataman的running为0，也就是没有在运行。

#### 串口连接很慢

使用串口连接差不多需要一分钟

主循环sleep(1)：telem2连接与usb连接差不多

主循环sleep(1)：telem2连接约30s，而usb连接差不多60s，且中间断过并重联3次

> 注意，地面站连接串口，启用流控的话，就无法获取参数。

### 后续优化

#### 通信端口优化

将串口、UDP等通信端口独立出来。创建MavlinkPort端口类，通过虚函数提供统一的接口，串口、UDP等通过继承该类并实现具体的通信。

```c++
class MavlinkPort{
public:
    MavlinkPort();
    virtual ~MavlinkPort();
    virtual int read(*buf) = 0;
    virtual int write(*buf, len) = 0;
};

class SerialPort : public MavlinkPort{
public:
    int read(*buf) override{
    }
    int write(*buf, len) override{
    }
};
```



## logger

### uorb

#### 构造函数

构造函数传参时，使用const修饰meta

是否需要增加一个使用ORB ID的构造函数

#### 需要增加几个函数

```c++
    bool valid() const {
        return _meta != nullptr;
    }
    const struct orb_metadata *get_topic() const {
        return _meta;
    }

    virtual bool subscribe() {
        _handle = orb_subscribe(_meta);
        return true;
    }
    virtual uint8_t get_instance() {
        return _instance;
    }
```

#### orb multi

需要实现orb multi

### 线程

#### poll事件

在log_writer_mavlink.cpp中，PX4的uorb的订阅可以触发poll事件，当前nextpolit的uorb机制是否支持该事件









# 问题

