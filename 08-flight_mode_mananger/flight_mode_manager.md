# flight mode manager

## 说明

flight_mode_manager用于多旋翼模态下给控制器提供local setpoints（位置、速度、加速度、油门期望值）。

在各种模式下生成setpoint，其输入是无人机当前模式、状态、Home点位置、triplets等，输出是提供给控制器的setpoints，也就是根据多旋翼飞行模式给控制器提供期望。为了使无人机飞行更流畅，避免“阶跃输入”导致姿态大角度，增加了控制指令平滑，例如对速度进行平滑。

> 在固定翼模态下不运行任何飞行任务，仅对多旋翼有效。
>
> 与navigator不同，flight mode manager的setpoint是local坐标系下的。

## 类继承关系

flight_mode_manager中核心是两个类：FlightModeManager（飞行任务管理）、FlightTask（飞行任务），其中为了针对不同飞行模式，FlightTask又生成了众多子类。

![flight_mode_manager_class](imgs\flight_mode_manager_class.svg)

各子类的简单介绍如下：

- FlighTaskAuto

  对应了所有自动飞行模式，如航线、悬停、起飞、降落等。

  - FlightTaskAutoMapper

    根据各自动飞行模式下各种不同的航点类型调用相应的函数生成setpoints；

  - FlightTaskAutoLineSmoothVel

    实现速度平滑；

## 流程框架

### 主循环流程

类FlightModeManager为本模块的核心类，用于管理所有飞行任务。主循环中周期性调用`FlightModeManager::update()`函数，并根据订阅vehicle_local_position来推动整个流程。

飞行任务管理器的update单次循环的流程框架如下：

![flight_mode_manager_frame](imgs\flight_mode_manager_frame.svg)

### setpoint流程

由FlightTask生成setpoints（包括`_position_setpoint`、`_velocity_setpoint`等），然后由FlightModeMananger调用FlightTask::getPositionSetpoint()获取后，发布至主题`ORB_ID(trajectory_setpoint)`。

这里`ORB_ID(trajectory_setpoint)`主题的消息类型本质为`vehicle_local_position_setpoint_s`。

![flight_mode_manager_setpoints](imgs\flight_mode_manager_setpoints.svg)

## 任务管理器-FlightModeManager

### 功能

- 订阅各类消息；

  订阅无人机状态（vehicle_status）、控制模式（vehicle_control_mode）等。

- 切换飞行任务：

  根据无人机状态指定要切换的飞行任务，如果当前已经存在一个飞行任务则通过”析构函数“删除然后再创建相应的飞行任务。

- 发布setpoints；

### 输入输出

#### 输入

- ORB_ID(vehicle_local_position)

  无人机local位置，设置为20ms更新一次。

- ORB_ID(vehicle_status)

  无人机状态

  消息发布模块：commander

- ORB_ID(home_position)

- ORB_ID(vehicle_control_mode)

  控制模式

  消息发布模块：commander

- ORB_ID(vehicle_command)

  无人机控制指令，如绕圆（轨道模式）等。

- ORB_ID(takeoff_status)

- ORB_ID(vehicle_land_detected)

#### 输出

- ORB_ID(trajectory_setpoint)

  输出位置、速度、角度期望值

  对应的订阅模块：fw_pos_control、mc_pos_control

  消息内容：vehicle_local_position_setpoint_s

- ORB_ID(vehicle_constraints)

  速度限幅包括水平速度、向上速度、向下速度；是否起飞。

- ORB_ID(vehicle_command)

  发布无人机控制指令，用于switchTask()失败后的处理。如果任务切换失败，则切换无人机飞行模式。

- ORB_ID(landing_gear)

  起落架

- ORB_ID(vehicle_command_ack)

  订阅到ORB_ID(vehicle_command)后进行回复

## 飞行任务-FlightTask

飞行任务最主要的就是运行三个函数：updateInitialize()、update()、bool updateFinalize()，分别是更新前初始化、更新、更新后限幅处理。





## 自主飞行任务-FlightTaskAuto

### 功能

- FlightTask：任务更新（包括更新前初始化、更新、更新后限幅处理）
- FlightTaskAutoMapper：根据各自动模式下对应的航点类型（WaypointType），对应的准备setpoints
- FlightTaskAutoLineSmoothVel：平滑速度

### 输入输出

#### 输入

- ORB_ID(position_setpoint_triplet)

  auto任务中使用。

### update流程

1. 起落架设置

2. 根据航点类型准备setpoints

3. 速度平滑

### 设置航点类型

当自动任务订阅到navigator发布的triplets信息，会根据triplets中current.type进行设置：

```c
_type = (WaypointType)_sub_triplet_setpoint.get().current.type;
```

航点类型种类如下：

```c
enum class WaypointType : int {
    position      = position_setpoint_s::SETPOINT_TYPE_POSITION,
    velocity      = position_setpoint_s::SETPOINT_TYPE_VELOCITY,
    loiter        = position_setpoint_s::SETPOINT_TYPE_LOITER,
    takeoff       = position_setpoint_s::SETPOINT_TYPE_TAKEOFF,
    land          = position_setpoint_s::SETPOINT_TYPE_LAND,
    idle          = position_setpoint_s::SETPOINT_TYPE_IDLE,
    follow_target = position_setpoint_s::SETPOINT_TYPE_FOLLOW_TARGET,
};
```

可以看到，这里的航点类型与triplets中的SETPOINT_TYPE一一对应。

### 生成setpoint

由FlightTaskAutoMapper根据航点类型，调用各类对应的`_prepareXXXXXSetpoints()`处理函数生成`_position_setpoint、_velocity_setpoint`。



## 类讲解

### FlightModeManager

重要成员函数

- generateTrajectorySetpoint()

  获取各飞行任务经过计算得到的setpoint（vehicle_local_position_setpoint），通过ORB_ID(trajectory_setpoint)发布。

### FlightTask

#### 重要成员函数

- const vehicle_local_position_setpoint_s getPositionSetpoint()

  各飞行任务经过计算得到了各种setpoint，通过该函数可以获取这些setpoint（vehicle_local_position_setpoint），该函数由FlightModeManager调用。

包括了如下：

```c
    matrix::Vector3f _position_setpoint;
    matrix::Vector3f _velocity_setpoint;
    matrix::Vector3f _velocity_setpoint_feedback;
    matrix::Vector3f _acceleration_setpoint;
    matrix::Vector3f _acceleration_setpoint_feedback;
    matrix::Vector3f _jerk_setpoint;
```



## 附录

消息内容：vehicle_local_position_setpoint_s

```c
	uint64_t timestamp;
	float x;
	float y;
	float z;
	float yaw;
	float yawspeed;
	float vx;
	float vy;
	float vz;
	float acceleration[3];
	float jerk[3];
	float thrust[3];
```

