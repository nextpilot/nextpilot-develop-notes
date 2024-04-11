# 固定翼offboard控制设计

## 介绍

### 控制能力说明

​		与多旋翼不同，固定翼无法悬停，必须持续飞行（要么绕圆盘旋），故控制逻辑有些不一样。其控制逻辑说明如下：

- 位置控制

  可选择控制global地理位置或者控制local位置，飞行过程中偏航角由期望位置计算得到，飞行到目标位置后绕圆盘旋。

- 速度控制

  可控制local NED速度，速度类型可选则空速或地速（默认空速），飞行过程中偏航角由期望速度水平分量计算得到。

- 位置和速度控制

  可以同时控制local NED下的位置和巡航速度，速度可以选择空速或地速，飞行过程中偏航角由期望位置计算得到。

- 高度控制

  可以控制高度和爬升/下降速率，高度类型可选择为相对home或海拔高度；如果**高度无效速率有效**则一直按照期望速率飞行，如果**高度有效速率无效**则以期望高度飞行，如果**二者都有效**，则高度期望做为限制，达到限制前以期望速率飞行。

  飞行过程中保持航向不变。

- 高度和速度控制

  可以控制水平速度和高度，也可以控制巡航速度和高度。

- 航向角控制

  可以控制航向角和航向角速率，如果**角度无效角速率有效**则一直按照期望角速率飞行，如果**角度有效角速率无效**则以期望角度飞行，且以转向最小方向调整偏航角，如果**二者都有效**，则角度期望做为限制，达到限制前以期望角速率飞行。

  飞行过程中保持高度不变（由于通过横滚来调整飞行航向，高度可能有一点下降）。

- 姿态控制

  可以控制固定翼姿态角和油门，注意，一定要设置油门量（0~1），不然无法飞行。

## 设计

### 输入输出

#### 输入

- 输入源：

  mavlink_receiver模块；

- 输入1：航迹数据
  - 订阅主题：ORB_ID(trajectory_setpoint)
  - 消息类型：vehicle_local_position_setpoint_s

#### 输出

- 输出至：

  fw_pos_control::control_position()函数，并由`ecl_l1_pos_controller _l1_control`和`TECS _tecs`进行处理;

- 输出1：current航点

  消息类型：position_setpoint_triplet_s.current，主要是设置如下几个结构体成员：

  - current.valid: true
  - current.type: position_setpoint_s::SETPOINT_TYPE_POSITION
  - current.lat: 设定航点纬度
  - current.lon: 设定航点经度
  - current.alt: 设定航点高度
  - current.cruising_speed: 设定巡航速度；
  - current.cruising_throttle: NAN，不控制油门；

- 输出2：previous航点

  消息类型：position_setpoint_triplet_s.current，主要是设置如下几个结构体成员：

  - previous.valid: true

  - previous.type: position_setpoint_s::SETPOINT_TYPE_POSITION

  - previous.lat: 设定航点纬度

  - previous.lon: 设定航点经度

  - previous.alt: 设定航点高度

- 输出3：横滚角

  将机载控制输入计算出一个期望航向角，进而计算横滚角并叠加至控制器（TECS/L1）的输出横滚角，或者将机载控制输入计算的横滚角替换控制器（TECS/L1）的输出横滚角。

### 处理逻辑

​		整个处理逻辑就是根据订阅的setpoint生成期望航点和调整横滚角。

#### 初始化

需要的相关飞行数据：

- 无人机经纬度、海拔高度；
- 航迹角：根据地速向量计算；
- 飞行速度（地速）：根据地速向量计算；
- 引导航点距离：无人机前后1km；

#### 设定航点

- 当期望航点有效（setpoint.x、setpoint.y）时

  直接以期望航点做为引导点，到达后绕圆。

- 当期望水平速度有效时

  根据水平地速速度分量计算当前航迹角，并沿着该航迹角，在无人机当前位置前面一定距离（如1km）设置current引导点，后面一定距离（如1km）设置previous引导点，由L1算法引导飞行。

- 其他

  根据水平地速速度分量计算当前拍航迹角并保存为一个默认值，并沿着该航迹角，在无人机当前位置前面一定距离（如1km）设置current引导点，后面一定距离（如1km）设置previous引导点，由L1算法引导飞行。
  
  注意，为了保证无人机一直飞直线，只需要计算首次进入该逻辑时的第一拍航迹角。

#### 设定高度

- 当期望爬升/下降速率setpoint.vz有效、期望高度有效时

  设定一个很高或很低的高度（根据vz方向），并设置爬升/下降速度参数来控制速率，当接近期望高度时，则将期望高度直接做为设定高度，保持该高度飞行。

- 当期望爬升/下降速率setpoint.vz有效时、期望高度无效时

  设定一个很高或很低的高度（根据vz方向），并设置爬升/下降速度参数来控制速率，一直以该速率爬升/下降。

- 当期望爬升/下降速率setpoint.vz无效时、期望高度有效时

  直接将期望高度直接做为设定高度，保持该高度飞行。
  
- 当期望爬升/下降速率setpoint.vz、期望高度都无效时

  获取当前一拍的当前高度作为期望高度。

  注意，高度不能给NAN。

#### 设定巡航速度

- 当水平期望速度setpoint.vx和setpoint.vy有效时

  巡航速度为水平期望速度的模，即$cruise=\sqrt{vx^2+vy^2}$。

- 当水平期望速度只有vx有效

  巡航速度即为vx。

- 当水平期望速度都无效

  巡航速度为默认值（FW_AIRSPD_TRIM）。

#### 调整横滚角（方式1）

​		飞行航向是由通过控制横滚角实现的。需要根据期望航向角计算得到横滚角调整量。

- 控制航向角速率

  根据角速率和飞行地速，计算出向心加速度，进而计算出横滚角，该横滚角将**替换**控制器（TECS/L1）的输出横滚角。

- 期望航点有效

  根据要飞到的期望航点计算航向角，该横滚角将**叠加**至控制器（TECS/L1）的输出横滚角。

- 期望速度

  根据水平期望速度计算航向角，该横滚角将**叠加**至控制器（TECS/L1）的输出横滚角。

#### 调整横滚角（方式2）

​		飞行航向是由通过控制横滚角实现的。需要根据期望航向角计算得到横滚角调整量。

- 控制航向角速率

  根据角速率和飞行地速，计算出向心加速度，进而计算出横滚角，该横滚角将**替换**控制器（TECS/L1）的输出横滚角。

- 期望航点有效

  根据设置期望航点为current航点，由L1算法引导飞行航向。

- 期望速度

  根据水平期望速度计算期望航迹角，并沿着该期望航迹角，在无人机当前位置前面一定距离（如1km）设置current引导点，后面一定距离（如1km）设置previous引导点，由L1算法引导飞行。。



### 失控保护

飞控从offboard异常退出，会出现failsafe："no RC and no offboard"。

该失控保护的应急处理由COM_OBL_ACT参数决定，默认是

> 机载计算机应该按照正常流程，调用SDK函数退出offboard模式，这样飞控不会发生failsafe。

## 新增参数说明

| 参数         | 功能                                                         | 默认值 | 定义文件           |
| ------------ | ------------------------------------------------------------ | ------ | ------------------ |
| OFFB_R_K     | 一个比例系数，根据航迹角偏差计算一个期望的横滚角             | 0.3    | commander_params.c |
| OFFB_ACC_YAW | 航迹角控制范围。当同时指定**期望角速率**和期望角度时，航迹角偏差小于这个范围时，只会根据期望航迹角进行响应，不再根据期望航迹角速率进行响应。<br>当航迹角测量值与期望值偏差较大时，如果指定了航迹角速率，则根据期望角速率控制航向，当偏差较小时，只根据期望角度控制航向了。<br>当航向测量精度较差时，建议增大该参数。 | 15     | commander_params.c |
| OFFB_ACC_ALT | 高度控制范围。当同时指定期望高度和期望天向速度时，当高度偏差小于这个范围时，只会响应高度，不再响应速度。<br>当高度测量值与期望值偏差较大时，如果指定了天向速度，则根据期天向速度控制高度，当偏差较小时，只根据期望高度控制高度了。<br/>当高度测量精度较差时，建议增大该参数。 |        | commander_params.c |
| OFFB_R_X_LIM | 横滚角修正值限幅，根据期望航向计算期望飞行横滚角后，通过限幅后再用该横滚角替换L1算法计算的横滚角。 |        | commander_params.c |
| OFFB_R_B_LIM | 横滚角补偿值限幅，根据期望航向计算期望飞行横滚角后，通过限幅后再将该横滚角作为补偿量与L1算法计算的横滚角进行叠加。 |        |                    |



## 控制算法

### 高度控制

给定期望高度后，赋值给`_pos_sp_triplet.current.alt`，并调用control_position()函数进行控制。



总能量控制

update_pitch_throttle()



## 失控保护相关

如果offboard模式下，外部控制数据断掉，则无人机会根据如下参数响应：

- COM_OBL_ACT：外部控制丢失且遥控器信号无效时的执行动作；推荐返航。
- COM_OBL_RC_ACT：外部控制丢失且遥控器信号有效时的执行动作；推荐悬停。

## 附录

### vehicle_local_position_setpoint_s

消息定义如下：

```c++
uint64 timestamp	# time since system start (microseconds)

uint8 SPEED_TYPE_AIR = 0
uint8 SPEED_TYPE_GROUND = 1

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
uint8 speed_type # speed type（新增）
```



### position_setpoint_triplet_s

消息定义如下：

```c++
uint64 timestamp		# time since system start (microseconds)

position_setpoint previous
position_setpoint current
position_setpoint next
    
```





### position_setpoint_s

消息定义如下：

```c++
uint64 timestamp		# time since system start (microseconds)

uint8 SETPOINT_TYPE_POSITION=0	# position setpoint
uint8 SETPOINT_TYPE_VELOCITY=1	# velocity setpoint
uint8 SETPOINT_TYPE_LOITER=2	# loiter setpoint
uint8 SETPOINT_TYPE_TAKEOFF=3	# takeoff setpoint
uint8 SETPOINT_TYPE_LAND=4	# land setpoint, altitude must be ignored, descend until landing
uint8 SETPOINT_TYPE_IDLE=5	# do nothing, switch off motors or keep at idle speed (MC)
uint8 SETPOINT_TYPE_FOLLOW_TARGET=6  # setpoint in NED frame (x, y, z, vx, vy, vz) set by follow target

uint8 VELOCITY_FRAME_LOCAL_NED = 1 # MAV_FRAME_LOCAL_NED
uint8 VELOCITY_FRAME_BODY_NED = 8 # MAV_FRAME_BODY_NED

bool valid			# true if setpoint is valid
uint8 type			# setpoint type to adjust behavior of position controller

float32 vx			# local velocity setpoint in m/s in NED
float32 vy			# local velocity setpoint in m/s in NED
float32 vz			# local velocity setpoint in m/s in NED
bool velocity_valid		# true if local velocity setpoint valid
uint8 velocity_frame		# to set velocity setpoints in NED or body
bool alt_valid		# do not set for 3D position control. Set to true if you want z-position control while doing vx,vy velocity control.

float64 lat			# latitude, in deg
float64 lon			# longitude, in deg
float32 alt			# altitude AMSL, in m
float32 yaw			# yaw (only for multirotors), in rad [-PI..PI), NaN = hold current yaw
bool yaw_valid			# true if yaw setpoint valid

float32 yawspeed		# yawspeed (only for multirotors, in rad/s)
bool yawspeed_valid		# true if yawspeed setpoint valid

int8 landing_gear		# landing gear: see definition of the states in landing_gear.msg

float32 loiter_radius		# loiter radius (only for fixed wing), in m
int8 loiter_direction		# loiter direction: 1 = CW, -1 = CCW

float32 acceptance_radius   # navigation acceptance_radius if we're doing waypoint navigation

float32 cruising_speed		# the generally desired cruising speed (not a hard constraint)
float32 cruising_throttle	# the generally desired cruising throttle (not a hard constraint)

bool disable_weather_vane   # VTOL: disable (in auto mode) the weather vane feature that turns the nose into the wind

bool rmb_tko                # Moving platform take-off flag
bool rmb_lnd                # Moving platform landing flag

```

