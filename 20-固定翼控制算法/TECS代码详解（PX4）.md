# TECS源码分析（PX4）

## 简介

TECS做为固定翼控制模块中的一个类，其提供了必要的函数接口以供调用。

TECS最重要的两个函数就是update_vehicle_state_estimates()、update_pitch_throttle()，前者获取无人机状态、指令输入并进行估计与更新，后置生成俯仰与油门指令。

## TECS类详解

### 主要函数成员变量

#### _SPE_estimate

无人机当前势能。specific potential energy estimate。

#### _SKE_estimate

无人机当前动能。

#### _SPE_rate

势能变化率，即一阶导数。specific potential energy rate estimate (m**2/sec**3)

#### _SKE_rate

动能变化率，即一阶导数。



#### _SPE_setpoint

无人机势能需求。specific potential energy demand (m^2/sec^2)。

#### _hgt_setpoint

期望高度，通过气压计进行初始化。

由updateHeightRateSetpoint()函数和_update_height_rate_setpoint()函数更改。

#### _hgt_rate_setpoint

期望天向速度。

由updateHeightRateSetpoint()函数和_update_height_rate_setpoint()函数更改。

### 函数讲解

#### update_vehicle_state_estimates()

更新无人机状态估计。

- 使用互补滤波器（一阶低通）对真空速速率进行滤波；

形参如下：

- equivalent_airspeed：无人机当前的当量空速；
- speed_deriv_forward：无人机机体下前向加速度，也就是acc_x，通过滤波获取真空速速率；
- altitude_lock：无人机是否进行高度控制；
- in_air：无人机是否在飞行中；
- altitude：无人机高度，海拔高度；
- vz：无人机天向速度（垂直方向速度）；



#### update_pitch_throttle()

TECS控制器更新函数，计算期望俯仰、油门。

- pitch：无人机俯仰角，一般使用真实俯仰角减去俯仰角配平值（FW_PSP_OFF）；

- baro_altitude：无人机当前海拔高度；

- hgt_setpoint：期望海拔高度；

- EAS_setpoint：期望当量空速，可以认为是期望指示空速；

- equivalent_airspeed：无人机当前的当量空速；

- eas_to_tas：当量空速转真空速，先使用1，如果TAS与CAS都有效，则使用TAS/CAS，该函数被调用前根据当前气压进行计算；

- climb_out_setpoint：是否希望急剧爬升，为true则进入ECL_TECS_MODE_CLIMBOUT模式，一般在滑跑起飞时有用；

- pitch_min_climbout：

- throttle_min：最小油门，根据参数FW_THR_MIN；

- throttle_max：最大油门，根据参数FW_THR_MAX；

- throttle_cruise：巡航油门，根据参数FW_THR_CRUISE；

- pitch_limit_min：FW_P_LIM_MIN；

- pitch_limit_max：FW_P_LIM_MAX；

- target_climbrate：FW_T_CLMB_R_SP

- target_sinkrate：FW_T_SINK_R_SP

- hgt_rate_sp：设置期望天向速度（爬升/下降率），默认值NAN；

  如果指定期望天向速度，则使用最大爬升率和最大下降率参数对其进行限幅；如果没有指定，则根据高度计算；



```c++
if (rt_isfinite(hgt_rate_sp)) {
    // 如果指定期望天向速度，则使用最大爬升率和最大下降率参数对其进行限幅
    // use the provided height rate setpoint instead of the height setpoint
    _update_height_rate_setpoint(hgt_rate_sp);
} else {
    // 若没有指定，则根据高度计算
    // calculate heigh rate setpoint based on altitude demand
    updateHeightRateSetpoint(hgt_setpoint, target_climbrate, target_sinkrate, baro_altitude);
}
```



#### _update_speed_states()

更新速度相关状态变量。通过二阶互补滤波器，计算_tas_state。

更新如下变量：

- _EAS_setpoint：期望当量空速，根据输入实参设置；
- _TAS_setpoint：望值真空速期，根据输入实参设置；
- _EAS：当量空速，根据输入实参设置；
- _tas_state：真空速，由滤波器；

#### _update_speed_height_weights()

​		更新分配比重。

- _SKE_weighting：速度（动能）比重，根据参数FW_T_SPDWEIGHT决定；

- _SPE_weighting：高度（势能）比重，\_SPE_weighting=2-\_SKE_weighting；

​		代码中，对分配比重进行了限幅，将两个比重都限制在[0,1]之间。

```c++
_SPE_weighting = constrain(2.0f - _SKE_weighting, 0.f, 1.f);
_SKE_weighting = constrain(_SKE_weighting, 0.f, 1.f);
```

注意，二者不是`你多我就少，你少我就多的关系`：

- 当速度比重\_SKE_weighting在[0,1]之间变化时，高度比重不变，\_SPE_weighting==1；
- 当速度比重\_SKE_weighting在[1,2]之间变化时相当于没变（\_SKE_weighting==1），而高度比重\_SPE_weighting对应变化；

#### _update_speed_setpoint()

更新如下变量：

- _TAS_setpoint

- _TAS_rate_setpoint

#### _update_height_rate_setpoint()

​		更新期望天向速度。

​		如果实参传入的天向速度有效则使用传入值；如果实参传入值无效，则通过`updateHeightRateSetpoint()`函数计算出一个天向速度。

#### updateHeightRateSetpoint()

根据期望高度以及当前高度，计算期望天向速度（_hgt_rate_setpoint）。

- alt_sp_amsl_m：设定飞行海拔高度；
- target_climbrate_m_s：目标爬升速率，由FW_T_CLMB_R_SP参数给定；
- target_sinkrate_m_s：目标下沉速率，由FW_T_SINK_R_SP参数给定；
- alt_amsl：当前飞行海拔高度；



默认情况下，没有指定期望天向速度时，则根据高度计算出期望天向速度。

1. 计算期望高度：_hgt_setpoint，根据给定的爬升/下沉速率与dt计算一个新的期望高度；
2. 计算前馈天向速度
3. 计算期望天向速度

_height_error_gain：高度差到天向速度P值，计算公式为\_height_error_gain=1.0/FW_T_ALT_TC，表示误差1米时需要的响应速度；

```c++
_hgt_rate_setpoint = (_hgt_setpoint - alt_amsl) * _height_error_gain + _height_setpoint_gain_ff * feedforward_height_rate;
```

其中：FW_T_ALT_TC默认为2，即如果误差是1m，希望以1/2=0.5m/s的速度调整。

4. 限幅

```c++
_hgt_rate_setpoint = constrain(_hgt_rate_setpoint, -_max_sink_rate, _max_climb_rate);
```



#### _update_energy_estimates()

更新能量估计。

- 计算当前势能、动能、总能量；

- 计算期望势能、期望动能、期望总能量；
- 计算能量差。

**无人机势能**

在物理学中，势能计算公式为$E=m*h*g$，在代码中，质量$m$可以通过归一化后不再考虑。无人机势能即：
$$
SPE=h*g
$$
**无人机动能**

在物理学中，动能计算公式为$E=\frac{1}{2}*m*v^2$，在代码中，质量$m$可以通过归一化后不再考虑。无人机动能即：
$$
SKE=\frac{1}{2}*v^2
$$
故能量的单位是$m^2/s^2$。



**势能期望**

势能期望由期望飞行高度决定。计算如下：
$$
SPE_{sp}=h_{sp}*g
$$

**动能期望**

动能期望由期望飞行速度决定（真空速）。计算如下：
$$
SKE_{sp}=0.5*TAS_{sp}*TAS_{sp}
$$
**总能量误差**

​		总能量误差为势能误差与动能误差之和。
$$
STE_{err}=SPE_{sp}-SPE_{est}+SKE_{sp}-SKE_{est}
$$
**能量平衡误差**

​		能量平衡误差为能量平衡期望与能量平衡做差。
$$
SEB_{err}=\\(SPE_{sp}*SPE_{weight}-SKE_{sp}*SKE_{weight})-\\
(SPE_{est}*SPE_{weight}-SKE_{est}*SKE_{weight})
$$

**势能变化率期望**

势能变化率期望由高度变化率决定。
$$
\dot{SPE_{sp}}=\dot{h_{sp}}*g
$$

- 势能变化率：\_SPE_rate_setpoint，即$\dot{SPE_{sp}}$；

- 高度变化率：\_hgt_rate_setpoint，即$\dot{h_{sp}}$；

**动能变化率期望**

动能变化率期望由空速决定。
$$
\dot{SKE_{sp}}=TAS*\dot{TAS_{rate_{sp}}}
$$

- 空速：\_tas_state，即$TAS$；
- 空速变化率：\_TAS_rate_setpoint，即$\dot{TAS_{rate_{sp}}}$；

**势能估计**

根据垂直方向位置（高度）进行估计。

```c++
_SPE_estimate = _vert_pos_state * CONSTANTS_ONE_G; // potential energy
```



**动能估计**

根据空速状态进行估计：

```c++
_SKE_estimate = 0.5f * _tas_state * _tas_state;    // kinetic energy
```



**势能变化率估计**

根据垂直方向速度进行估计：

```c++
_SPE_rate = _vert_vel_state * CONSTANTS_ONE_G; // potential energy rate of change
```



**动能变化率估计**

根据空速以及空速变化率进行估计：

```c++
_SKE_rate = _tas_state * _tas_rate_filtered;   // kinetic energy rate of change
```



#### _update_throttle_setpoint()

根据总能量需求计算期望油门。

**计算总能量变化率**

1. 计算变化率

   ```c+++
   float STE_rate_setpoint = _SPE_rate_setpoint + _SKE_rate_setpoint;
   ```

   

2. 通过\_STE_rate_max与\_STE_rate_min限制总能量变化率。

   ```c++
   STE_rate_setpoint = constrain(STE_rate_setpoint, _STE_rate_min, _STE_rate_max);
   ```

   - \_STE_rate_max：表示以最大爬升率（最大油门条件下）飞行时总能量的变化率；
   - \_STE_rate_min：表示以最小下沉率（最小油门条件下）飞行时总能量变化率；
   
   二者在`TECS::_update_STE_rate_lim()`函数中计算如下：
   
   ```c++
   _STE_rate_max = _max_climb_rate * CONSTANTS_ONE_G;
   _STE_rate_min = -_min_sink_rate * CONSTANTS_ONE_G;
   ```
   
   在这里`_max_climb_rate`对应的参数是FW_T_CLMB_MAX，`_min_sink_rate`对应的参数是FW_T_SINK_MIN。



**预估一个油门（比例控制）**

在巡航油门基础上，根据能量变化率预估一个油门，当期望能量变化率是增大时，就在巡航油门和最大油门之间计算，当期望能量变化率是减小时，就在最小油门和巡航油门之间计算。

```c++
if (STE_rate_setpoint >= 0) { // 油门在巡航值与最大值之间
    throttle_predicted = throttle_cruise + STE_rate_setpoint / _STE_rate_max * (_throttle_setpoint_max - throttle_cruise);
} else { // 油门在最小值与巡航值之间
    throttle_predicted = throttle_cruise + STE_rate_setpoint / _STE_rate_min * (_throttle_setpoint_min - throttle_cruise);
}
```

**计算期望油门（微分控制）**

根据总能量变化率误差进行微分项调整

```c++
throttle_setpoint = (_STE_rate_error * _throttle_damping_gain) * STE_rate_to_throttle + throttle_predicted;
```

**调整期望油门（积分控制）**

当空速有效时，通过积分项调整油门，消除稳态误差。

```c++
float throttle_integ_input = (_STE_rate_error * _integrator_gain_throttle) * _dt * STE_rate_to_throttle;
```

进行调整。

```c++
throttle_setpoint += _throttle_integ_state;
```



**其他**

- 限制总能量变化率

通过\_STE_rate_max与\_STE_rate_min限制总能量变化率，



- 油门变化需求

​		油门的变化需求与高度变化率（爬升率）以及速度变化有关。其中对于爬升率来说：

如果指定了爬升率则通过`_update_height_rate_setpoint()`函数进行设定，如果未指定爬升率则通过`updateHeightRateSetpoint()`函数进行计算。



#### _update_pitch_setpoint()

根据能量平衡需求计算期望俯仰角。



计算能量平衡变化率误差（_SEB_rate_error）

```c++
_SEB_rate_error = SEB_rate_setpoint - (_SPE_rate * _SPE_weighting - _SKE_rate * _SKE_weighting);
```

计算积分项（_pitch_integ_state）

```c++
float pitch_integ_input = _SEB_rate_error * _integrator_gain_pitch;

_pitch_integ_state = _pitch_integ_state + pitch_integ_input * _dt;
```

> _integrator_gain_pitch： I增益



爬升角（俯仰角）与能量平衡变化率认为是线性关系，即速度与重力加速度的乘积，故有：

```c++
const float climb_angle_to_SEB_rate = _tas_state * CONSTANTS_ONE_G;
```

> 也就是说，可以认为climb_angle_to_SEB_rate是一个系数：
>
> SEB_rate = climb_angle_to_SEB_rate * climb_angle
>
> climb_angle = SEB_rate / climb_angle_to_SEB_rate

俯仰角期望就可以通过能量平衡变化率得到：

```c++
_pitch_setpoint_unc = SEB_rate_correction / climb_angle_to_SEB_rate;
```



计算能量平衡变化率修正量

```c++
float SEB_rate_correction = _SEB_rate_error * _pitch_damping_gain + _pitch_integ_state + _SEB_rate_ff * SEB_rate_setpoint;
```

> _pitch_damping_gain：P增益

##### 理论

根据平衡能量定义：
$$
SEB=mgh-\frac{1}{2}mV^2
$$
短时间内俯仰角的变化会使势能与动能相互转化，根据势能与动能的转化关系，势能的增加代表动能的减少，反之亦然。故势能的变化率与动能变化率绝对值一样但符号相反。
$$
\dot{SEB}=mg\dot{h}-mV\dot{V}\\
=mgV_T-mV\dot{V}\\
=mgVsin(\theta)-mV\dot{V}\\
=m*V*g*sin(\theta)-m*V*\dot{V}
$$
在代码中，认为势能的变化率就是能量平衡的变化率也是可以的。
$$
\dot{SEB}=m*V*g*sin(\theta)
$$
通过质量归一化$m=1$，且俯仰角小角度时认为$sin(\theta)=\theta$，则有俯仰角与能量平衡变化率的关系：
$$
\dot{SEB}=V*g*\theta
$$
那么俯仰角就可以通过能量变化率得到：
$$
\theta=\frac{\dot{SEB}}{V*g}
$$
