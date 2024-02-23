## system

### 仿真时间获取问题

问题描述：

程序运行1个多小时后，获取的当前时间是个很小的值。刚开始时间是正确的。

deadline: 4294968000, now: 704, peroid: 4000

分析：

rt_tick_get()返回uint32_t类型，再乘以1000可能导致溢出。

```c++
hrt_abstime hrt_absolute_time(void) {
#else
    return rt_tick_get() * (1000000 / RT_TICK_PER_SECOND);
#endif
}
```

原因：

仿真时获取时间返回值溢出

## mavlink

[508] I/mav_main: mavlink mavlink start -m0 -r115200 -t10.0.2.2 -o14550 -u18570
[532] I/mav_main: mode: Normal(115200), data rate: 18570 B/s on udp port 14550 remote port 8995

远端端口错误









