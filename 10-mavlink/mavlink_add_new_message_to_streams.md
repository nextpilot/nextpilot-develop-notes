# 添加新消息至下发列表

## 概述

​		本章节描述如何将新增的ORB消息至下发列表，进而可以发送至地面站。

## 具体步骤

在mavlink/streams中增加ENGINE_STATUS.hpp





mavlink_messages.cpp

包含新增的头文件

```c
#include "streams/ENGINE_STATUS.hpp"
```

下发列表新增一条

```c
static const StreamListItem streams_list[] = {
#if defined(ENGINE_STATUS_HPP)
    create_stream_list_item<MavlinkStreamEngineStatus>()
#endif // ENGINE_STATUS_HPP
};
```



mavlink_main.cpp

在对应模式配置中增加下发消息

```c
int Mavlink::configure_streams_to_default(const char *configure_single_stream) {
    switch (_mode) {
    case MAVLINK_MODE_NORMAL:
        configure_stream_local("ENGINE_STATUS", 5.0f);
    }
}
```

