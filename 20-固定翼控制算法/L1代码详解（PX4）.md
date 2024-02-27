# L1算法代码详解（PX4）



## 框架





## 类ecl_l1_pos_controller

### 成员变量

#### _L1_distance

L1点距离，通过公式_L1_distance = _L1_ratio * ground_speed得到。

### 成员函数

#### update_roll_setpoint()

​		根据向心加速度计算横滚角期望。

#### get_roll_setpoint()

​		获取计算出的横滚角期望。



#### nav_bearing()

​		获取计算出的偏航角期望。

```c
    float nav_bearing() {
        return matrix::wrap_pi(_nav_bearing);
    }
```



#### navigate_waypoints

航线引导

**参数**

- vector_A：一般为上一个期望点（经纬度）；
- vector_B：当前期望点（经纬度）；
- vector_curr_position：当前无人机位置（经纬度）；
- ground_speed_vector：无人机地速向量；



**代码**

- 计算到目标的航向

  ```c
  _target_bearing = get_bearing_to_next_waypoint(vector_curr_position(0), vector_curr_position(1), vector_B(0), vector_B(1));
  ```

- 计算L1

  ```c
  _L1_distance = _L1_ratio * ground_speed;
  ```

- 计算向量AB

  计算由A到B的向量，并归一化。

  ```c
  Vector2f vector_AB = get_local_planar_vector(vector_A, vector_B);
  vector_AB.normalize();
  ```

- 计算向量AP

  计算由A到vector_curr_position的向量。

  ```c
  Vector2f vector_A_to_airplane = get_local_planar_vector(vector_A, vector_curr_position);
  ```

- 计算AB与AP向量形成的夹角

  



## 代码详解



### 航线飞行流程解析

806行navigate_waypoints()

```c
else if (position_sp_type == position_setpoint_s::SETPOINT_TYPE_POSITION ||
                 (_vehicle_status.is_vtol && position_sp_type == position_setpoint_s::SETPOINT_TYPE_LAND)) {
	_l1_control.navigate_waypoints(prev_wp, curr_wp, curr_pos, nav_speed_2d);
    
}
```



## 测试记录

无人机任务规划为：起飞、给定垂直起飞点、设置第一个航点、设置第二、三、四个航点。

#### 到达起飞高度



#### 旋翼转成固定翼的那刻

- prev_wp:即起飞点的经纬度

29.3826605，

104.5789349

- curr_wp:在垂直转换点东边999781m

29.5292583

94.2485931

#### 固定翼飞行垂直转换点

- prev_wp:上一个阶段的curr_wp

29.5292583，

94.2485931

- curr_wp:垂直转换点

29.3824168

104.5766502



#### 到达转换点飞向第一个航点

- prev_wp:上一个阶段的curr_wp（垂直转换点）

29.3824168，

104.5766502

- curr_wp:第一个航点

29.3808915

104.5723864

#### 到达第一个航点飞向第二个航点

- prev_wp:上一个阶段的curr_wp（第一个航点）

29.3808915

104.5723864

- curr_wp:第二个航点

29.3763471

104.5716148



- pos_sp_next：表示下一个航点（也就是第三个航点）

29.3753760

104.5774822



### 指点飞行

830行navigate_loiter()

```c++
else if (position_sp_type == position_setpoint_s::SETPOINT_TYPE_LOITER) {
    _l1_control.navigate_loiter(curr_wp, curr_pos, loiter_radius, loiter_direction, nav_speed_2d);
}
```



- prev_wp:无效会设置成curr_wp



- curr_wp:指点位置经纬度

29.3783963

104.5752986



- pos_sp_prev: 发送指点飞行时无人机所在位置





### offboard-前期

当进入offboard后，只给current航点赋值。

pos_sp_prev: 0

pos_sp_curr: offboard赋值

pos_sp_next: 0



```c++
_l1_control.navigate_waypoints(prev_wp, curr_wp, curr_pos, nav_speed_2d);
```



previous航点会设置为current一样的经纬度

prev_wp: 29.3833374, 104.5890717



### offboard-修改无法保持当前航向的问题

当不指定航迹角时，记录当前拍航迹角，沿着航迹角在无人机当前位置前后1km生成引导航点，后方为previous航点，前方为current航点。

prev_wp: 29.387360613184097, 104.58542414393699

curr_wp: 29.374925858686648, 104.57051011648359



