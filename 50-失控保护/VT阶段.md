# VT阶段失控保护





## 遥控器模式切换

commander可以根据遥控器模式通道进行模式切换，在commander中，通过`main_state_transition()`函数进行控制模式切换，在使用遥控器进行模式切换时调用流程如下：

```flow
st=>start: Commander::run()
op1=>operation: set_main_state()
op2=>operation: set_main_state_rc()
op3=>operation: main_state_transition()
st()->op1()->op2()->op3()
```









```flow
st=>start: 开始框
op=>operation: 处理框
cond=>condition: 判断框(是或否?)
sub1=>subroutine: 子流程
io=>inputoutput: 输入输出框
e=>end: 结束框
st(right)->op(right)->cond
cond(yes)->io(bottom)->e
cond(no)->sub1(right)->op
```

_vtol_vehicle_status.vtol_transition_failsafe = true;





