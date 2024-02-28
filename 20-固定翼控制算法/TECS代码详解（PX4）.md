# TECS源码分析（PX4）



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

#### _update_energy_estimates

更新能量估计。当前势能、动能、总能量、期望势能、期望动能、期望总能量，以及能量差。

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



#### _update_throttle_setpoint

根据总能量需求计算期望油门。



#### _update_pitch_setpoint

根据能量平衡需求计算期望俯仰角。



#### update_pitch_throttle

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





_update_height_rate_setpoint



#### updateHeightRateSetpoint

更新期望高度以及期望天向速度。

默认情况下，没有指定期望天向速度时，则根据高度计算出期望天向速度。