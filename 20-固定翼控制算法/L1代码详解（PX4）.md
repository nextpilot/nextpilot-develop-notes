# L1算法代码详解（PX4）



## 框架



## 类fw_pos_control

### 航线飞行流程解析

806行navigate_waypoints()

```c
else if (position_sp_type == position_setpoint_s::SETPOINT_TYPE_POSITION ||
                 (_vehicle_status.is_vtol && position_sp_type == position_setpoint_s::SETPOINT_TYPE_LAND)) {
	_l1_control.navigate_waypoints(prev_wp, curr_wp, curr_pos, nav_speed_2d);
}
```



## 类ecl_l1_pos_controller

### 成员变量

#### _L1_distance

​		L1点距离，通过公式_L1_distance = _L1_ratio * ground_speed得到。

### 成员函数

#### update_roll_setpoint()

​		根据向心加速度计算横滚角期望。

#### get_roll_setpoint()

​		获取计算出的横滚角期望。

```c++
float get_roll_setpoint() {
    return _roll_setpoint;
}
```



#### nav_bearing()

​		获取计算出的偏航角期望。

```c
float nav_bearing() {
    return matrix::wrap_pi(_nav_bearing);
}
```



#### navigate_waypoints

航线引导。根据两个引导点，控制无人机沿着两点连线飞行。

##### 参数

- vector_A：一般为上一个期望点（经纬度）；
- vector_B：当前期望点（经纬度）；
- vector_curr_position：当前无人机位置（经纬度），使用P表示；
- ground_speed_vector：无人机地速向量；

##### 代码解析

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

- 计算向量AP长度以及AP在AB上的投影长度

  这两个长度可用于计算AP与AP的夹角
  
  ```c++
  float distance_A_to_airplane = vector_A_to_airplane.length();
  float alongTrackDist         = vector_A_to_airplane * vector_AB;
  ```
  
- 计算向量BP

  计算向量BP并归一化
  
  ```c++
  Vector2f vector_B_to_P_unit = get_local_planar_vector(vector_B, vector_curr_position).normalized();
  ```
  
- 计算AB与BP向量形成的夹角
  
  根据向量叉积与点积运算，计算AB与BP的夹角

  ```c++
  float AB_to_BP_bearing = atan2f(vector_B_to_P_unit % vector_AB, vector_B_to_P_unit * vector_AB);
  ```
  
- 情况一：
  
  ```c++
  
  ```
  
- 情况二：

  ```c++
  ```
  
  
  
- 情况三：P在A、B点之间

  ```c++
  /* 计算踩着AB连线飞行的eta角， */
  
  /* 计算地速与向量V_AB叉积，得到sin */
  float xtrack_vel = ground_speed_vector % vector_AB;
  
  /* 计算地速与向量V_AB点积，得到cos */
  float ltrack_vel = ground_speed_vector * vector_AB;
  
  /* 计算eta2，地速与向量V_AB的夹角 */
  float eta2 = atan2f(xtrack_vel, ltrack_vel);
  
  /* 计算无人机P点到连线AB的垂直距离，叉积得到向量AP与AB围成的四边形面积，由于向量AP归一化后，|AB|=1，故四边形底长度=1，那么四边形面积就是高，也就是P点到连线AB的垂直距离 */
  float xtrackErr = vector_A_to_airplane % vector_AB;
  /* sin(eta1)=d/L1，其中d=高，L1为斜边 */
  float sine_eta1 = xtrackErr / math::max(_L1_distance, 0.1f);
  
  /* 限制eta在±45 degrees */
  sine_eta1  = math::constrain(sine_eta1, -0.7071f, 0.7071f); // sin(pi/4) = 0.7071
  float eta1 = asinf(sine_eta1);
  /* 得到eta */
  eta        = eta1 + eta2;
  
  /* 得到偏航角 */
  _nav_bearing = atan2f(vector_AB(1), vector_AB(0)) + eta1;
  ```

  





## 测试记录

### 航线测试

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





