# offboard

## 介绍

​		飞控计算能力和功能有限，为了扩展飞控功能，需要对外提供了控制接口，故飞控引入了offboard，使用offboard可实现外部设备发送控制信号进而完成更复杂的飞行控制任务，例如基于视觉的复杂自主导航功能。

## 基本功能

​		以下是v12.3版本实现的功能。

### 多旋翼

​		多旋翼具备悬停功能，故能够实现所有飞行控制能力。

- 位置控制

  可以控制Global和Local下的位置、航向角；

- 速度控制

  可以控制Local和Body下的速度、航向角；

- 姿态控制

  可以控制姿态角、油门量。

### 固定翼

​		与多旋翼不同，固定翼无法悬停，必须持续飞行（要么绕圆盘旋），故控制逻辑有些不一样。

- 位置控制

  可以通过发送经纬度坐标（SET_POSITION_TARGET_GLOBAL_INT ）或局部坐标（SET_POSITION_TARGET_LOCAL_NED ）控制指令设置固定翼飞行目标位置，飞行过程中目前无法控制偏航角和速度，飞行到目标位置后绕圆盘旋；

- 速度控制

  无法控制速度；

- 姿态控制

  可以通过发送SET_ATTITUDE_TARGET 控制指令设置固定翼姿态角和油门，注意，一定要设置油门量（0~1），不然无法飞行；

## 如何使用

​		机载计算机基于MAVLink通信协议通过串口/网口连接飞控后，需要以一定频率周期性发送控制信号，确定飞控收到控制信号后再设置飞控模式为offboard即可。

## 逻辑流程详解

### onboard计算机到飞控mavlink模块

​		机载计算机要发送的控制信号对应的消息主要有：

- MAV_CMD_DO_SET_MODE ([176](https://mavlink.io/en/messages/common.html#MAV_CMD_DO_SET_MODE) )

  设置模式，用于开启offboard飞行模式。

- SET_POSITION_TARGET_LOCAL_NED ([ #84 ](https://mavlink.io/en/messages/common.html#SET_POSITION_TARGET_LOCAL_NED))

  设置local或body下的期望位置、速度、加速度、偏航角等。

- SET_POSITION_TARGET_GLOBAL_INT ([ #86 ](https://mavlink.io/en/messages/common.html#SET_POSITION_TARGET_GLOBAL_INT))

  设置global下的期望位置、速度、加速度、偏航角等。
  
- SET_ATTITUDE_TARGET ([ #82 ](http://mav-dev.cetcs.com/en/messages/common.html#SET_ATTITUDE_TARGET))

  设置机体坐标系下的期望姿态角/角速度/推力。

### mavlink模块处理

​		在mavlink接收线程，对接收的消息进行解析并处理，转成对应的ORB消息发布，然后由对应的模块订阅接收。

1. 模式设置

![flow_set_mode](imgs\flow_set_mode.svg)

2. 位置控制信号

   ![flow_local_position](imgs\flow_local_position.svg)

3. 姿态控制信号

   ![flow_attitude](imgs\flow_attitude.svg)

​		

如果是VTOL机型。则发布姿态控制信号逻辑如下：

- 当前是多旋翼模态：发布主题ORB_ID(mc_virtual_attitude_setpoint)，由vtol_att_control订阅；
- 当前是固定翼模态：发布主题ORB_ID(fw_virtual_attitude_setpoint)，由vtol_att_control订阅；

#### 转ORB消息至commander模块

​		接收的控制信号（位置控制、姿态控制等消息）后会生成ORB_ID(offboard_control_mode)主题并发布offboard_control_mode_s消息。

​		commander通过Commander::offboard_control_update()订阅到该消息后，主要进行offboard模式有效性判断。

​		offboard_control_mode_s结构体包括了position、velocity、acceleration、attitude、body_rate五个布尔成员变量，表示是否需要控制对应物理量：

- position：true表示控制无人机位置
- velocity：true表示控制无人机速度

- acceleration：true表示控制无人机加速度

- attitude：true表示控制无人机姿态角

- body_rate：true表示控制无人机角速率

#### mavlink模块到控制模块

​		接收的控制信号（位置控制、姿态控制等消息）后会应生成ORB主题消息如下：

- 位置控制信号：ORB_ID(trajectory_setpoint)，发布vehicle_local_position_setpoint_s消息。
- 姿态控制信号：ORB_ID(vehicle_attitude_setpoint)，发布vehicle_attitude_setpoint_s消息。



> 如果mavlink收到的是GLOBAL期望则会根据无人机当前位置转成LOCAL期望。

### commander模块处理

#### offboard控制信号有效性判断

​		offboard模式的保持需要用户一直给飞控控制信号，如果信号中断那么飞控会从offboard模式退出。

​		commander判断offboard控制信号有效性，判断offoard无效的逻辑为：

- 当一定时间内没有订阅到ORB_ID(offboard_control_mode)主题消息或者消息内控制量表示无效则代表控制信号丢失，则offboard无效；

- 当订阅到的offboard_control_mode消息内五个控制标志量都为false时表示offboard无效；

	当控制信号丢失，会将内部变量**_status_flags.offboard_control_signal_lost**置false。

​		可以通过参数[COM_OF_LOSS_T](#COM_OF_LOSS_T)设置控制信号丢失时间判断，当该时间段内未收到控制信号则认为信号丢失。



#### 判断可否进入offboard模式的逻辑

​		通过发送set_mode命令至飞控，实现offboard的模式切换。飞控在切换前会进行一系列逻辑判断，当满足条件时才能够进入offboard模式。核心是判断

offboard控制信号是否丢失（offboard_control_signal_lost），当信号没有丢失时才能够进入offboard模式（status_flags.offboard_control_signal_lost=false）。

```flow
start=>start: commander主循环
run()  
receive=>operation: commander接收到offboard模式设置指令
handle_command()
transition=>operation: 进入状态转换处理函数
main_state_transition()
cond1_status=>condition: 判断offboard信号是否丢失
offboard_control_signal_lost
offboard=>operation: main_state=MAIN_STATE_OFFBOARD
set_nav_state=>operation: 进入导航状态设置处理函数
set_nav_state()
cond2_status=>condition: 判断offboard信号是否丢失
offboard_control_signal_lost
nav_state=>operation: nav_state=NAVIGATION_STATE_OFFBOARD
failsafe=>operation: 进入failsafe

start->receive
receive->transition
transition->cond1_status
cond1_status(no)->offboard
offboard->set_nav_state
set_nav_state->cond2_status
cond2_status(no)->nav_state
cond2_status(yes)->failsafe
```

​		故如果希望进入offboard模式，需要先发一次控制命令（例如SET_POSITION_TARGET_LOCAL_NED,#84)，用于表示offboard信号有效，然后再发送offboard模式设置命令（VEHICLE_CMD_SET_MODE, 176）。

#### 发布控制模式

​		commander在`Commander::update_control_mode()`函数中根据导航状态（`vehicle_status_s`），发布控制模式`ORB_ID(vehicle_control_mode)`。

​		如果已经进入了offboard模式，则会根据`ORB_ID(offboard_control_mode)`的控制量标志，则会创建vehicle_control_mode_s类型消息并发布。

### 控制量标志说明

#### onboard计算机至mavlink模块

​		onboard计算机至mavlink模块使用的是mavlink消息，在消息中通过`type_mask`字段表示哪些控制量无效；

#### mavlink模块至控制模块

​		mavlink模块至位置/姿态控制模块，是生成`vehicle_local_position_setpoint_s`变量并通过主题`ORB_ID(trajectory_setpoint)`发送，该变量内字段通过设置NAN表示对应控制量无效。

​		该变量内的字段将根据对应`type_mask`控制量是否无效进行设置，例如位置x控制无效：

```
setpoint.x = (type_mask & POSITION_TARGET_TYPEMASK_X_IGNORE) ? (float)NAN : target_local_ned.x;
```

​		控制模块订阅主题`ORB_ID(trajectory_setpoint)`接收到`vehicle_local_position_setpoint_s`变量数据后，也是根据对应字段是否为NAN来判断控制量是否有效。



### 扩展TYPEMASK

​		使用POSITION_TARGET_TYPEMASK表示各种控制量的有效性。

|      | Value | Field Name                                                   | Description                                            |
| ---- | ----- | ------------------------------------------------------------ | ------------------------------------------------------ |
| 0    | 1     | [POSITION_TARGET_TYPEMASK_X_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_X_IGNORE) | Ignore position x                                      |
| 1    | 2     | [POSITION_TARGET_TYPEMASK_Y_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_Y_IGNORE) | Ignore position y                                      |
| 2    | 4     | [POSITION_TARGET_TYPEMASK_Z_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_Z_IGNORE) | Ignore position z                                      |
| 3    | 8     | [POSITION_TARGET_TYPEMASK_VX_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_VX_IGNORE) | Ignore velocity x                                      |
| 4    | 16    | [POSITION_TARGET_TYPEMASK_VY_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_VY_IGNORE) | Ignore velocity y                                      |
| 5    | 32    | [POSITION_TARGET_TYPEMASK_VZ_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_VZ_IGNORE) | Ignore velocity z                                      |
| 6    | 64    | [POSITION_TARGET_TYPEMASK_AX_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_AX_IGNORE) | Ignore acceleration x                                  |
| 7    | 128   | [POSITION_TARGET_TYPEMASK_AY_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_AY_IGNORE) | Ignore acceleration y                                  |
| 8    | 256   | [POSITION_TARGET_TYPEMASK_AZ_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_AZ_IGNORE) | Ignore acceleration z                                  |
| 9    | 512   | [POSITION_TARGET_TYPEMASK_FORCE_SET](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_FORCE_SET) | Use force instead of acceleration                      |
| 10   | 1024  | [POSITION_TARGET_TYPEMASK_YAW_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_YAW_IGNORE) | Ignore yaw                                             |
| 11   | 2048  | [POSITION_TARGET_TYPEMASK_YAW_RATE_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_YAW_RATE_IGNORE) | Ignore yaw rate                                        |
| 12   |       |                                                              |                                                        |
| 13   |       | 用于表示期望速度类型                                         | 0：地速，1：空速；                                     |
| 14   |       | 用于表示期望航迹角速率类型                                   | 0：有方向角速率，1：无方向绝对角速率（按最短路径调整） |
| 15   |       |                                                              |                                                        |

### 扩展Frame

- MAV_FRAME_LOCAL_NED
  - 高度类型为Offboard::AltitudeType::RelHome
- MAV_FRAME_LOCAL_OFFSET_NED
  - 高度类型为Offboard::AltitudeType::Amsl

### 无人机控制处理

#### 多旋翼位置控制

MulticopterPositionControl.cpp



#### 固定翼位置控制

在fw_pos_control::update()函数，如果当前是固定翼模态且使能了offboard则订阅ORB_ID(trajectory_setpoint)并处理：

```c++
/**
 *
 */
if (_control_mode.flag_control_offboard_enabled && (_vehicle_status.vehicle_type == vehicle_status_s::VEHICLE_TYPE_FIXED_WING)) {
    // Convert Local setpoints to global setpoints
    if (!map_projection_initialized(&_global_local_proj_ref) || (_global_local_proj_ref.timestamp != _local_pos.ref_timestamp)) {
        map_projection_init_timestamped(&_global_local_proj_ref, _local_pos.ref_lat, _local_pos.ref_lon,
                                        _local_pos.ref_timestamp);
        _global_local_alt0 = _local_pos.ref_alt;
    }

    vehicle_local_position_setpoint_s trajectory_setpoint;

    if (_trajectory_setpoint_sub.update(&trajectory_setpoint)) {
        if (rt_isfinite(trajectory_setpoint.x) && rt_isfinite(trajectory_setpoint.y) && rt_isfinite(trajectory_setpoint.z)) {
            double lat;
            double lon;

            if (map_projection_reproject(&_global_local_proj_ref, trajectory_setpoint.x, trajectory_setpoint.y, &lat, &lon) == 0) {
                _pos_sp_triplet = {}; // clear any existing

                _pos_sp_triplet.timestamp                 = trajectory_setpoint.timestamp;
                _pos_sp_triplet.current.timestamp         = trajectory_setpoint.timestamp;
                _pos_sp_triplet.current.valid             = true;
                _pos_sp_triplet.current.type              = position_setpoint_s::SETPOINT_TYPE_POSITION;
                _pos_sp_triplet.current.lat               = lat;
                _pos_sp_triplet.current.lon               = lon;
                _pos_sp_triplet.current.alt               = _global_local_alt0 - trajectory_setpoint.z;
                _pos_sp_triplet.current.cruising_speed    = NAN; // ignored
                _pos_sp_triplet.current.cruising_throttle = NAN; // ignored
            }

        } else {
            mavlink_log_critical(&_mavlink_log_pub, "Invalid offboard setpoint");
        }
    }

} else {
    if (_pos_sp_triplet_sub.update(&_pos_sp_triplet)) {
        // reset the altitude foh (first order hold) logic
        _min_current_sp_distance_xy = FLT_MAX;
    }
}
```



## 参数

| 参数          |      | 说明                         |
| ------------- | ---- | ---------------------------- |
| COM_OF_LOSS_T |      | offboard指令判断丢失时间阈值 |
|               |      |                              |
|               |      |                              |

## 备注

### 相关mavlink消息

#### POSITION_TARGET_TYPEMASK

| Value | Field Name                                                   | Description                       |
| ----- | ------------------------------------------------------------ | --------------------------------- |
| 1     | [POSITION_TARGET_TYPEMASK_X_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_X_IGNORE) | Ignore position x                 |
| 2     | [POSITION_TARGET_TYPEMASK_Y_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_Y_IGNORE) | Ignore position y                 |
| 4     | [POSITION_TARGET_TYPEMASK_Z_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_Z_IGNORE) | Ignore position z                 |
| 8     | [POSITION_TARGET_TYPEMASK_VX_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_VX_IGNORE) | Ignore velocity x                 |
| 16    | [POSITION_TARGET_TYPEMASK_VY_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_VY_IGNORE) | Ignore velocity y                 |
| 32    | [POSITION_TARGET_TYPEMASK_VZ_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_VZ_IGNORE) | Ignore velocity z                 |
| 64    | [POSITION_TARGET_TYPEMASK_AX_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_AX_IGNORE) | Ignore acceleration x             |
| 128   | [POSITION_TARGET_TYPEMASK_AY_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_AY_IGNORE) | Ignore acceleration y             |
| 256   | [POSITION_TARGET_TYPEMASK_AZ_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_AZ_IGNORE) | Ignore acceleration z             |
| 512   | [POSITION_TARGET_TYPEMASK_FORCE_SET](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_FORCE_SET) | Use force instead of acceleration |
| 1024  | [POSITION_TARGET_TYPEMASK_YAW_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_YAW_IGNORE) | Ignore yaw                        |
| 2048  | [POSITION_TARGET_TYPEMASK_YAW_RATE_IGNORE](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK_YAW_RATE_IGNORE) | Ignore yaw rate                   |

#### SET_POSITION_TARGET_LOCAL_NED ([ #84 ](https://mavlink.io/en/messages/common.html#SET_POSITION_TARGET_LOCAL_NED))

| Field Name       | Type     | Units | Values                                                       | Description                                                  |
| ---------------- | -------- | ----- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| time_boot_ms     | uint32_t | ms    |                                                              | Timestamp (time since system boot).                          |
| target_system    | uint8_t  |       |                                                              | System ID                                                    |
| target_component | uint8_t  |       |                                                              | Component ID                                                 |
| coordinate_frame | uint8_t  |       | [MAV_FRAME](https://mavlink.io/en/messages/common.html#MAV_FRAME) | Valid options are: [MAV_FRAME_LOCAL_NED](https://mavlink.io/en/messages/common.html#MAV_FRAME_LOCAL_NED) = 1, [MAV_FRAME_LOCAL_OFFSET_NED](https://mavlink.io/en/messages/common.html#MAV_FRAME_LOCAL_OFFSET_NED) = 7, [MAV_FRAME_BODY_NED](https://mavlink.io/en/messages/common.html#MAV_FRAME_BODY_NED) = 8, [MAV_FRAME_BODY_OFFSET_NED](https://mavlink.io/en/messages/common.html#MAV_FRAME_BODY_OFFSET_NED) = 9 |
| type_mask        | uint16_t |       | [POSITION_TARGET_TYPEMASK](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK) | Bitmap to indicate which dimensions should be ignored by the vehicle. |
| x                | float    | m     |                                                              | X Position in NED frame                                      |
| y                | float    | m     |                                                              | Y Position in NED frame                                      |
| z                | float    | m     |                                                              | Z Position in NED frame (note, altitude is negative in NED)  |
| vx               | float    | m/s   |                                                              | X velocity in NED frame                                      |
| vy               | float    | m/s   |                                                              | Y velocity in NED frame                                      |
| vz               | float    | m/s   |                                                              | Z velocity in NED frame                                      |
| afx              | float    | m/s/s |                                                              | X acceleration or force (if bit 10 of type_mask is set) in NED frame in meter / s^2 or N |
| afy              | float    | m/s/s |                                                              | Y acceleration or force (if bit 10 of type_mask is set) in NED frame in meter / s^2 or N |
| afz              | float    | m/s/s |                                                              | Z acceleration or force (if bit 10 of type_mask is set) in NED frame in meter / s^2 or N |
| yaw              | float    | rad   |                                                              | yaw setpoint                                                 |
| yaw_rate         | float    | rad/s |                                                              | yaw rate setpoint                                            |

#### SET_POSITION_TARGET_GLOBAL_INT ([ #86 ](https://mavlink.io/en/messages/common.html#SET_POSITION_TARGET_GLOBAL_INT))

| Field Name       | Type     | Units | Values                                                       | Description                                                  |
| ---------------- | -------- | ----- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| time_boot_ms     | uint32_t | ms    |                                                              | Timestamp (time since system boot). The rationale for the timestamp in the setpoint is to allow the system to compensate for the transport delay of the setpoint. This allows the system to compensate processing latency. |
| target_system    | uint8_t  |       |                                                              | System ID                                                    |
| target_component | uint8_t  |       |                                                              | Component ID                                                 |
| coordinate_frame | uint8_t  |       | [MAV_FRAME](https://mavlink.io/en/messages/common.html#MAV_FRAME) | Valid options are: [MAV_FRAME_GLOBAL_INT](https://mavlink.io/en/messages/common.html#MAV_FRAME_GLOBAL_INT) = 5, [MAV_FRAME_GLOBAL_RELATIVE_ALT_INT](https://mavlink.io/en/messages/common.html#MAV_FRAME_GLOBAL_RELATIVE_ALT_INT) = 6, [MAV_FRAME_GLOBAL_TERRAIN_ALT_INT](https://mavlink.io/en/messages/common.html#MAV_FRAME_GLOBAL_TERRAIN_ALT_INT) = 11 |
| type_mask        | uint16_t |       | [POSITION_TARGET_TYPEMASK](https://mavlink.io/en/messages/common.html#POSITION_TARGET_TYPEMASK) | Bitmap to indicate which dimensions should be ignored by the vehicle. |
| lat_int          | int32_t  | degE7 |                                                              | X Position in WGS84 frame                                    |
| lon_int          | int32_t  | degE7 |                                                              | Y Position in WGS84 frame                                    |
| alt              | float    | m     |                                                              | Altitude (MSL, Relative to home, or AGL - depending on frame) |
| vx               | float    | m/s   |                                                              | X velocity in NED frame                                      |
| vy               | float    | m/s   |                                                              | Y velocity in NED frame                                      |
| vz               | float    | m/s   |                                                              | Z velocity in NED frame                                      |
| afx              | float    | m/s/s |                                                              | X acceleration or force (if bit 10 of type_mask is set) in NED frame in meter / s^2 or N |
| afy              | float    | m/s/s |                                                              | Y acceleration or force (if bit 10 of type_mask is set) in NED frame in meter / s^2 or N |
| afz              | float    | m/s/s |                                                              | Z acceleration or force (if bit 10 of type_mask is set) in NED frame in meter / s^2 or N |
| yaw              | float    | rad   |                                                              | yaw setpoint                                                 |
| yaw_rate         | float    | rad/s |                                                              | yaw rate setpoint                                            |



### 相关orb消息

#### offboard_control_mode.msg

```shell
# Off-board control mode

uint64 timestamp		# time since system start (microseconds)

bool position
bool velocity
bool acceleration
bool attitude
bool body_rate
```





#### vehicle_local_position_setpoint.msg

```shell
# Local position setpoint in NED frame
# setting something to NaN means the state should not be controlled

uint64 timestamp	# time since system start (microseconds)

float32 x		# in meters NED
float32 y		# in meters NED
float32 z		# in meters NED
float32 yaw		# in radians NED -PI..+PI
float32 yawspeed	# in radians/sec
float32 vx		# in meters/sec
float32 vy		# in meters/sec
float32 vz		# in meters/sec
float32[3] acceleration # in meters/sec^2
float32[3] jerk # in meters/sec^3
float32[3] thrust	# normalized thrust vector in NED

# TOPICS vehicle_local_position_setpoint trajectory_setpoint

```



