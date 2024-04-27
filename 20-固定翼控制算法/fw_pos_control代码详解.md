# 代码详解



fw_pos_control类：

主要函数

- init()：初始化函数，调用一次；
- update()：更新函数，每10ms运行一次；

主要成员

- _tecs：TECS实例，用于总能量控制；
- _l1_control：ecl_l1_pos_controller实例，用于L1位置控制；

### control_position()



### tecs_update_pitch_throttle()

- 调用_tecs.update_vehicle_state_estimates()
- 调用_tecs.update_pitch_throttle()







## 参数

| 参数名     | 说明         | 备注                                 |
| ---------- | ------------ | ------------------------------------ |
| FW_PSP_OFF | 俯仰角配平值 | 一般等于正常速度巡航时无人机俯仰角度 |
|            |              |                                      |
|            |              |                                      |

