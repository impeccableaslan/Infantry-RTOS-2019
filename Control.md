# 控制

## 模式切换

### 相关线程任务

chassis_task, gimbal_task, shoot_task
在这些task中，系统将根据遥控器左右拨杆的位置进入不同的模式。

**【注意】** 不同于ICRA2018，2019版本中没有相应的模式代号，诸如AUTO_MODE，亦没有专用的mode switch task。

#### 底盘模式切换

* 位于chassis_task中，利用RC的S2拨杆的开关情况和*键盘输入*进行切换

* 使用`rc_device_get_state(rc_dev, state)`函数对遥控器拨杆值进行判别：
    1. param-1：`rc_dev`是输入的遥控信号，这里使用`prc_dev`
    2. param-2：`state`是要侦测的值，例如`RC_S2_DOWN`
    3. returned value：一个int32_t值，若为`RM_OK`则表示相应开关已经触发

* 使用 **_TODO_** 函数对键盘和鼠标输入值进行判别
    1. **_TODO_**

* 当处在**底盘跟随云台**模式时，利用`pid_calculate`计算底盘转角

* 当不处在**底盘失能**模式时，利用`chassis_set_offset`和`chassis_set_speed`输出底盘控制指令

* 当拨杆正在切换时，例如RC_S2_MID2UP，底盘失能，利用`chassis_set_speed`向底盘输出0

* 可以通过chassis_set_acc设定底盘加速度，但是目前的手动控制中未被使用到

* 所有下行控制指令和底盘硬件反馈值均在`pchassis`结构体当中，其中反馈值被`chassis_imu_update`更新

* 所有遥控信号均储存在`prc_dev`结构体当中；所有上位机控制信号均储存在 **_TODO_** 结构体当中

#### 云台模式切换

* 位于gimbal_task中，利用RC的S1、S2的开关情况和*键盘、鼠标的输入*进行切换

* 使用`rc_device_get_state(rc_dev, state)`函数对遥控器拨杆值进行判别：
    与底盘相同

* 使用 **_TODO_** 函数对键盘和鼠标的输入进行判别
    1. **_TODO_**

* 当处在**底盘跟随云台**模式时，利用`gimbal_set_pitch_delta`输出俯仰（pitch）控制指令（此处为变化值，即delta）

* 利用`gimbal_set_yaw_angle`输出偏转（yaw）控制指令
    - 第二个参数为角度值，其零点为初始角度或者底盘角度

* 通过`gimbal_set_yaw_mode(pgimbal, MODE)`设定yaw控制指令是以初始角度为参照还是以底盘角度为参照
    - `GYRO_MODE`意味着以初始角度为0，云台按陀螺仪修正
    - `ENCODER_MODE`意味着以底盘指向为0，云台按yaw轴电机返回值修正

* 除控制指令外，此task亦包括：
    - imu恒温
        `imu_temp_ctrl_init()`和`imu_temp_ctrl_keep()`
    - 通过CAN向底盘主板云台控制信息
        `send_gimbal_curreent(iq1,iq2, iq3)`
    - 自动标定yaw、pitch零参位
        `auto_gimbal_adjust(pgimbal)`和`gimbal_auto_adjust_start()`
    - 初始化gimbal控制指令和反馈值，按照零参位对yaw、pitch轴电机归零
        `gimbal_state_init(pgimbal)`，`gimbal_init_state_reset()`和`get_gimbal_init_state()`

* 所有下行控制指令和底盘硬件反馈值均在`pchassis`结构体当中，其中反馈值被`chassis_imu_update`更新

* 所有遥控信号均储存在`prc_dev`结构体当中；所有上位机控制信号均储存在 **_TODO_** 结构体当中

#### 射击模式切换

* 位于shoot_task.c中，利用RC的S1开关，*拨轮*实现控制

* 启动/关闭摩擦轮       `shoot_firction_toggle(pshoot)`
* 射击若干发子弹        `shoot_set_cmd(pshoot, CMD, num_of_bullet)`
* 向射击执行机构发送命令 `shoot_execute(pshoot)`

* 使用`rc_device_get_state(rc_dev, state)`函数对遥控器拨杆值进行判别：
    与底盘相同

* 使用 **_TODO_** 函数对键盘和鼠标的输入进行判别
    1. **_TODO_**

* 所有下行控制指令和底盘硬件反馈值均在`pshoot`结构体当中

* 被调用的命令均储存在shoot.c之中，包括：
    - 外部调用：`shoot_set_fric_speed`   设定摩擦轮转速
    - 外部调用：`shoot_set_cmd`          设置射击指令
    - 外部调用：`shoot_execute`，其调用了：
        - `shoot_fric_ctrl`，其调用了：
            - `shoot_get_fric_speed`    获取左右摩擦轮转速
            - `fric_set_output`         设定摩擦轮电机电流
            - 并：平衡两侧转速
        - `shoot_block_check`           利用电流检查是否卡弹
        - `shoot_cmd_ctrl`，其调用了：
            - `shoot_state_update`      利用~~触动开关~~ **_trigger motor_** 返回值检查是否有一发子弹射出，
            并进行射击保险
            - 并：检查射出子弹数、进行state transfer
    - 外部调用：`shoot_enable`
    - 外部调用：`shoot_disable`

### 控制信号流向

#### 接收器
采用DMA方式，值将保存在buff中，供下一步处理

#### 信号处理

基本原理：
* 初始化过程中将rc device注册，绑定实现相应的update过程的成员方法
* 执行过程中：调用`rc_device_data_update`函数以获取遥控信息
* 执行过程中：调用`rc_device_get_state`函数以获取提取出遥控信息中的状态
* 执行过程中：调用`rc_device_data_info`函数以获取提取出遥控信息中的数据

init.c中调用 `rc_device_register(rc_dev, name, flags)` ———— 初始化
board.c或communicate.c中调用（取决于gimbal还是chassis）
            `rc_device_data_update(rc_dev, buff)` ———— 更新数据，其调用了
                `rc_dev->get_data(rc_dev, buff)` -> `get_dr16_data(rc_dev, buff)` ———— 更新数据
                `rc_dev->get_state(rc_dev)`      -> `get_dr16_state(rc_dev)`      ———— 从更新的数据中提取flag信息
相应task调用 `rc_device_get_state(rc_dev, state)` ———— 判定对应标志位是否值为真
相应task调用 `rc_device_get_info(rc_dev)`         ———— rc_info的getter

**【注意】**
* rc结构体包含：
    - rc_info       各信道、键位的值
    - last_rc_info  各信道、键位的先前值
    - state         flags，仅限于S1和S2的当前，和变化（例如，S2从Mid至Down——RC_S2_MID2DOWN）
* 由于采用了oop的编程思想，info和state的两个成员经由对应的getter获取

#### 控制解算
在chassis或者gimbal的task中实现，参见上一节
###
---


## 操作指南
### 模式

模式由右拨杆控制：
1. 右拨杆【上】底盘跟随云台
2. 右拨杆【中】失能
3. 右拨杆【下】云台跟随底盘

### 控制

#### 摩擦轮和弹仓盖（英雄机器人：直线电机）

* 由左拨杆触发：
1. 左拨杆【由中至上】开启/关闭摩擦轮
2. 左拨杆【由中至下】开启/关闭弹仓盖，或伸出/缩回直线电机

- 触发指令后，请将左拨杆归中
- 摩擦轮和激光同步开启关闭，不打开摩擦轮不能射击

* 由键盘触发：
1. 按下 `F` 键，开启摩擦轮
2. 按下 `F+Z` 键，关闭摩擦轮

3. 按下 `R` 键，打开弹仓盖/伸出直线电机
4. 按下 `R+Z` 键，关闭弹仓盖/缩回直线电机

#### 射击

* 由拨轮触发
1. 上推拨轮：连发
2. 下拉拨轮：单发

* 由鼠标触发
1. 单击鼠标左键：单发
2. 长按鼠标左键：连发
3. 长按鼠标右键：追踪装甲

#### 避弹

按住`V`键

- 步兵采用自旋避弹
- 英雄采用扭腰避弹

#### 驾驶

* 移动

1. 遥控左操纵杆
2. 键盘`W``A``S``D`
3. 低速：`Ctrl`
4. 高速: `Shift`

* 转动和俯仰

1. 遥控器右操纵杆
2. 鼠标
3. 左右转动：`Q``E`
