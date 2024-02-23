# 固定翼控制



## 函数

### control_position

根据三点期望（position_setpoint_triplet_s）和当前无人机状态计算期望姿态角

获取position_setpoint_triplet_s消息有两种方式：

- 通过直接订阅；
- 在offboard模式下根据外部输入指令进行计算；



### 获取roll和yaw期望

通过L1控制。

```c++
_l1_control.navigate_waypoints(prev_wp, curr_wp, curr_pos, nav_speed_2d);
_att_sp.roll_body = _l1_control.get_roll_setpoint();
_att_sp.yaw_body  = _l1_control.nav_bearing();
```





# 不指定高度

## 仿真

737行函数，get_distance_to_point_global_wgs84()函数返回值为NAN，dist_z=NAN



在updateHeightRateSetpoint()中，alt_sp_amsl_m=NAN，三个逻辑都没进。计算出的_hgt_rate_setpoint是比较小的值



### 记录

**进入offboard的打印输出**

```shell
fw-1: 42.00, nan
TECS-3-0: 3.00, 2.00, 0.00, 0.08
TECS-3-0: 1109916586, 1109916586, 0, 1033476506
TECS-3-1: 42.00, 42.00, 0.00, 0.08
TECS-1: 42.00, 42.00, 43.08, 0.20, 0.30, 0.00
TECS-2: -0.22
fw-1: nan, nan
TECS-3-0: 3.00, 2.00, nan, 0.08
TECS-3-0: 2143289344, 1109916586, 2143289344, 1033476506
TECS-1: nan, 42.00, 43.08, 0.20, 0.30, 0.00
TECS-2: -0.22
[119591] I/commander: handle command 176
[119592] W/commander: switch into offboard flight mode
[119619] I/navigator: handle vehicle cmd=176
fw-1: nan, nan
TECS-3-0: 3.00, 2.00, nan, 0.08
TECS-3-0: 2143289344, 1109916586, 2143289344, 1033476506
TECS-1: nan, 42.00, 43.08, 0.20, 0.30, 0.00
TECS-2: -0.22
```





## 真机

fw_pos_control.cpp

810：position_sp_alt=0.07

2020：alt_sp=-0.02

tecs.cpp

160：alt_sp_amsl_m=cannot evaluate， _hgt_setpoint=nan



### 分析

进入updateHeightRateSetpoint()函数后，各传入参数如下：

- alt_sp_amsl_m=nan，期望高度
- alt_amsl=614.0，当前高度
- target_climbrate_m_s=3，爬升率
- target_sinkrate_m_s=3，下沉率

各变量如下

- _dt=0.025



```c++
// target_climbrate_m_s=3,
target_climbrate_m_s = min(target_climbrate_m_s, _max_climb_rate);
// target_sinkrate_m_s=nan
target_sinkrate_m_s  = min(target_sinkrate_m_s, _max_sink_rate);


// 该条件成立
// fabsf(alt_sp_amsl_m - _hgt_setpoint) = nan
// max(target_sinkrate_m_s, target_climbrate_m_s) * _dt=0.08
if (fabsf(alt_sp_amsl_m - _hgt_setpoint) < max(target_sinkrate_m_s, target_climbrate_m_s) * _dt) {
        rt_kprintf("TECS-3-1: %.2f, %.2f, %.2f, %.2f\n", alt_sp_amsl_m, _hgt_setpoint, fabsf(alt_sp_amsl_m - _hgt_setpoint), max(target_sinkrate_m_s, target_climbrate_m_s) * _dt);
        _hgt_setpoint = alt_sp_amsl_m;// _hgt_setpoint=nan

    } 

// _hgt_rate_setpoint=nan
_hgt_rate_setpoint = (_hgt_setpoint - alt_amsl) * _height_error_gain + _height_setpoint_gain_ff * feedforward_height_rate;
//_hgt_rate_setpoint=5
_hgt_rate_setpoint = constrain(_hgt_rate_setpoint, -_max_sink_rate, _max_climb_rate);
```







增加了一些打印，结果却不一样了。在constrain函数前多增加_hgt_rate_setpoint的打印，_hgt_rate_setpoint的结果不再是5，而是0了。

```c
// 造成结果是5
//rt_kprintf("TECS-1: %.2f, %.2f, %.2f, %.2f, %.2f, %.2f\n", alt_sp_amsl_m, _hgt_setpoint, alt_amsl, _height_error_gain, _height_setpoint_gain_ff, feedforward_height_rate);

// 造成结果是0
    rt_kprintf("TECS-1: %.2f,, %.2f, %.2f, %.2f, %.2f, %.2f, %.2f\n", _hgt_rate_setpoint, alt_sp_amsl_m, _hgt_setpoint, alt_amsl, _height_error_gain, _height_setpoint_gain_ff, feedforward_height_rate);

// 最后结果
_hgt_rate_setpoint = constrain(_hgt_rate_setpoint, -_max_sink_rate, _max_climb_rate);
```





### 记录

**进入offboard的打印输出**

```shell
01-31 18:32:08.117 I/comfw-1: 391.24, -inf
TECS-3-0: 3.00, 2.00, 0.00, 0.08
TECS-3-0: 1136893580, 1136893580, 0, 1033477311
TECS-3-1: 391.24, 391.24, 0.00, 0.08
TECS-1: 391.24, 391.24, 391.87, 0.20, 0.30, 0.00
TECS-2: -0.13
mander: handle command 176
0fw-1: -inf, -inf
TECS-3-0: 3.00, 2.00, -inf, 0.07
TECS-3-0: 2143289344, 1136893580, 2143289344, 1033413692
TECS-3-1: -inf, 391.24, -inf, 0.07
TECS-1: -inf, -inf, 391.87, 0.20, 0.30, 0.00
TECS-2: 5.00
1-31 18:32:08.117 W/commander: switch intfw-1: -inf, -inf
TECS-3-0: 3.00, 2.00, -inf, 0.08
TECS-3-0: 2143289344, 2143289344, 2143289344, 1033476908
TECS-3-1: -inf, -inf, -inf, 0.08
TECS-1: -inf, -inf, 391.87, 0.20, 0.30, 0.00
TECS-2: 5.00
o offboard flight mode
01-31 18fw-1: -inf, -inf
TECS-3-0: 3.00, 2.00, -inf, 0.07
## alt_sp_amsl_m, _hgt_setpoint, aa, bb
TECS-3-0: 2143289344, 2143289344, 2143289344, 1033464829
## alt_sp_amsl_m, _hgt_setpoint, aa, bb
TECS-3-1: -inf, -inf, -inf, 0.07
TECS-1: -inf, -inf, 391.87, 0.20, 0.30, 0.00
TECS-2: 5.00
:32:08.191 I/navigator: handle vehicle cfw-1: -inf, -inf
TECS-3-0: 3.00, 2.00, -inf, 0.08
TECS-3-0: 2143289344, 2143289344, 2143289344, 1033476908
TECS-3-1: -inf, -inf, -inf, 0.08
TECS-1: -inf, -inf, 391.88, 0.20, 0.30, 0.00
TECS-2: 5.00
md=176
fw-1: -inf, -inf
TECS-3-0: 3.00, 2.00, -inf, 0.08
TECS-3-0: 2143289344, 2143289344, 2143289344, 1033487780
TECS-3-1: -inf, -inf, -inf, 0.08
TECS-1: -inf, -inf, 391.88, 0.20, 0.30, 0.00
TECS-2: 5.00
```



## 结论

当不指定高度时，alt_sp_amsl_m=NAN，`_hgt_setpoint`仍然是上一拍的有效值。

故计算fabsf(alt_sp_amsl_m - _hgt_setpoint)=NAN。

关于NAN的逻辑比较判断：

- 在keil中

if(-inf<0.08)=true

- 在gcc中

if(nan<0.08)=false



所以对于如下逻辑的判断在gcc和keil下的判断有差异：

```c
fabsf(alt_sp_amsl_m - _hgt_setpoint) < max(target_sinkrate_m_s, target_climbrate_m_s) * _dt
```

- 在keil编译环境下，判断为true
- 在gcc编译环境下，判断为false

**keil编译环境**

这样对于会进入如下逻辑

```c
if (fabsf(alt_sp_amsl_m - _hgt_setpoint) < max(target_sinkrate_m_s, target_climbrate_m_s) * _dt) {
        _hgt_setpoint = alt_sp_amsl_m;
}
```

使得`_hgt_setpoint`赋值为NAN，导致计算使用如下代码计算的`_hgt_rate_setpoint`也是NAN。

```c
_hgt_rate_setpoint = (_hgt_setpoint - alt_amsl) * 		
    _height_error_gain + _height_setpoint_gain_ff * 	
    feedforward_height_rate;
```

`_hgt_rate_setpoint`成为NAN并经过constrain限幅函数后变成了5。

**gcc编译环境**

不会使得`_hgt_setpoint`赋值为NAN，`_hgt_setpoint`就会一直保持进入offboard前面一拍的有效值，这样计算出的`_hgt_rate_setpoint`就不是NAN，通过constrain限幅函数后仍然是接近0的速度。



# 附录

在keil下通过浮点打印NAN，打印结果是`-inf`，在gcc下打印结果是`nan`，通过无符号32位整型打印都是2143289344。
