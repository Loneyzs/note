代码组织、运行层面，两部分。举例理解：

开发人形机器人的运动控制部分： 把电机等底层硬件，开发组织成节点，对外提供通讯接口，然后用户需要使用的时候就通过自己写节点发布通讯。

```
电机驱动/编码器/IMU：
        ↓
硬件接口节点：读取编码器、发送电流/速度/位置指令。
        ↓
关节控制节点：完成 PID、限位、状态反馈。
        ↓
全身运动控制节点：计算各关节目标，例如步态、平衡、逆运动学。
        ↓
任务/行为节点：发送“向前走”“抬手”“目标速度”等命令。
```

对外暴露节点通信接口。（ros2本身更偏向于“功能支持”的开发）

内部通信可通过代码耦合方式，或也使用ros2通信实现。



# 基本单元

## 项目结构

```
 pytros2_ws/            <-- 运行 colcon build 的地方（工作空间根目录）
├── build/              <-- 编译产物（自动生成）
├── install/            <-- 运行产物（自动生成，以后 source 这里的 setup.bash）
├── log/                <-- 日志（自动生成）
└── src/
    ├── package_1/
    │   ├── package.xml
    │   ├── setup.py
    │   ├── setup.cfg
    │   ├── resource/
    │   │   └── package_1
    │   ├── test/
    │   │   ├── test_copyright.py
    │   │   ├── test_flake8.py
    │   │   └── test_pep257.py
    │   └── package_1/
    │       ├── __init__.py
    │       ├── node_1.py
    │       ├── node_2.py
    │       ├── utils.py
    │       └── config_loader.py
    │
    ├── package_2/
    │   ├── package.xml
    │   ├── setup.py
    │   ├── setup.cfg
    │   ├── resource/
    │   │   └── package_2
    │   └── package_2/
    │       ├── __init__.py
    │       ├── node_3.py
    │       ├── node_4.py
    │       └── algorithm.py
    │
    └── package_3/
        ├── package.xml
        ├── setup.py
        ├── setup.cfg
        ├── resource/
        │   └── package_3
        ├── launch/
        │   └── bringup.launch.py
 
        └── package_3/
            ├── __init__.py
            └── helper_node.py
工作空间中包含多个功能包，每个功能包包含多个节点
```

## 节点

执行功能的最小单元，通过Node类实现

每个 node 都有自己的生命周期，运行时节点之间通过 topic/service/action 相互调用，实现类似os效果

开发接口 && 工作流：

* ```python
  from rclpy.node import Node
  node = Node('node_name')
  node.get_logger().info('') 	#输出调试信息
  rclpy.spin(node) 			#堵塞执行节点
  rclpy.shutdown()
  ```

* `ros2 node info` 查看节点信息

* `ros2 node list` 查看节点列表

## 功能包

代码组织形式，一般实现一个功能会同时运行多个节点，同时承担了工程模块封装的作用。

相当于把多个完整程序组合到了一起。

开发接口 && 工作流：

1. 在工作空间`/src` 中使用 `ros2 pkg create --build-type ament_python <package_name>` 初始化功能包

2. 在 `my_package/my_package/ ` 中开发代码

3. 在package.xml中添加依赖 `<depend>xx<depend>` ，若包之间有依赖顺序，也可以在这里加。

4. 在setup.py中添加每个节点的main入口：

```python
  entry_points={
        'console_scripts': [
            '可执行文件名字（某个节点） = my_package_name.文件名:main',
        ],
```

5. 在工作空间根目录运行 `colcon build --symlink-install` 全部编译 或  `colcon build --packages-select my_package` 单独编译某个功能包
6. 在工作空间根目录运行  `source install/setup.bash` 自动配置运行的环境变量set
7. 运行：`ros2 run <package_name> <executable_name>` 启动编译对应节点

## launch启动脚本

用于同时启动多个节点

文件结构

```
没有统一要求，可以单独写一个启动功能包，也可放在某个功能包下
humanoid_ws/
└── src/
    ├── humanoid_hardware/
    ├── humanoid_control/
    ├── humanoid_perception/
    ├── humanoid_interfaces/
    └── humanoid_bringup/
        ├── launch/
        │   ├── hardware.launch.py
        │   ├── control.launch.py
        │   ├── perception.launch.py
        │   └── bringup.launch.py
        └── config/
            ├── motor.yaml
            ├── control.yaml
            └── vision.yaml
            
my_package/
├── launch/
│   └── demo.launch.py
├── my_package/
│   ├── node_1.py
│   ├── node_2.py
│   └── node_3.py
├── package.xml
└── setup.py
```

简单伪代码

```python
# bringup.launch.py

def generate_launch_description():
    # 生成一份“启动说明书”
    node_1 = Node(
        # 定义一个动作：启动某个节点
        package="pkg",
        executable="node_1",
        name="node_1"
    )

    node_2 = Node(
        package="pkg",
        executable="node_2",
        name="node_2"
    )

    node_3 = Node(
        package="pkg",
        executable="node_3",
        name="node_3"
    )

    return LaunchDescription([
        # 把多个动作打包成完整启动方案
        node_1,
        node_2,
        node_3
    ])

```

`setup.py` 安装 launch 文件

```python
import os
from glob import glob

data_files=[
    ('share/ament_index/resource_index/packages',
        ['resource/' + package_name]),
    ('share/' + package_name, ['package.xml']),

    # 安装 launch 文件
    (os.path.join('share', package_name, 'launch'), glob('launch/*.launch.py')),
],
```

使用 launch 启动：`ros2 launch humanoid_bringup bringup.launch.py` 





# 通信

ros2多节点同时运行，相互之间依赖调用通过通信进行，本质是在传统接口上多写了一层通信控制，回调调用。

单元测试仍通过传统方式直接调用api进行。

## 话题通信

**1.概念：**

节点发布话题，另外节点们订阅（广播）

`ros2 topic info 话题名称` 查看某个话题信息

`ros2 topic echo 话题名称` 查看某个话题具体值

`ros2 topic pub 话题名称 消息接口 数据内容(yaml`)  命令行发布话题，其中消息接口为类似int等固定类型

**2.话题发布伪代码：**

```python
from example_interfaces.msg import XXX #添加example_interfaces依赖

# 直接发布
self.my_publisher = self.creae_publisher(Type, 'Topic_Name', 队列长度)
self.my_publisher.publish(msg)

# 回调发布（定时等场景）
class ImuNode:
    def __init__(self):
        self.imu = ImuDriver()

        # 创建发布者
        self.pub = create_publisher(
            topic="/imu/data",
            msg_type=Imu
        )

        # 每 10ms 调用一次 timer_callback
        self.timer = create_timer(
            period=0.01,
            callback=self.timer_callback
        )

    def timer_callback(self):
        data = self.imu.read()

        msg = Imu()
        msg.acc = data.acc
        msg.gyro = data.gyro

        self.pub.publish(msg)
        # 节点运行后开始调用
```

**3.话题订阅伪代码：**

```python
# motor_node.py

class MotorNode:
    def __init__(self):
        self.motor = MotorDriver()

        # 创建订阅 /motor_cmd, 只要有新消息，就自动调用 self.cmd_callback
        self.sub = create_subscription(
            topic="/motor_cmd",
            msg_type=Int32,
            callback=self.cmd_callback
        )

    def cmd_callback(self, msg):
        speed = msg.data
        self.motor.set_speed(speed)
        
        # 节点运行后开始调用
```

**4.自定义接口**

使用 专用接口包 实现

1. 创建专门的接口包`ws/interface_pkg/`

   * 不使用 `src/` 文件夹
   * 根目录创建 `msg/`: 创建.msg文件放话题数据定义

   * 根目录创建`srv/`: 创建.srv文件放服务请求/响应定义

2. .xml中进行声明`<member_of_group>rosidl_interface_packages</member_of_group>`

​	其他需要使用该服务接口的包在.xml中引入该包依赖

## 服务通信

**1.概念：**

两节点间进行双向通信，是有结果的话题（本质是两个话题）

`ros2 service list`

`ros2 service call 服务名称 消息接口(类型) 请求内容(yaml)`

`rqt ` GUI操作(plugin中)

**2.客户端伪代码**

发布请求并接受反馈结果

```python
class DecisionNode:
    def __init__(self):
        self.client = create_client(
            srv_type=SetBool,
            service_name="/set_motor_enable"
        )

    def enable_motor(self):
        request = SetBool.Request()
        request.data = True

        response = self.client.call(request)

        if response.success:
            print("电机使能成功")
```

**3.服务端伪代码**

负责执行请求并反馈

```python
class MotorNode:
    def __init__(self):
        self.srv = create_service(
            srv_type=SetBool,
            service_name="/set_motor_enable",
            callback=self.enable_callback
        )

    def enable_callback(self, request, response):
        if request.data == True:
            self.motor.enable()
            response.success = True
            response.message = "motor enabled"
        else:
            self.motor.disable()
            response.success = True
            response.message = "motor disabled"

        return response
```

## 参数通信

**1.概念**

参数属于某个节点本身

`ros2 param list` 查看参数列表

`ros2 param describe / get / set 节点名称 参数名称` 获取/设置参数

**2.设置参数**

大多数参数都是静态的，配置在对应功能包 `config/node_name.yaml` 下

```python
# config/motor.yaml

motor_node:
  ros__parameters:
    motor_id: 1
    max_speed: 6000
    kp: 1.2
    ki: 0.1
```

节点中声明并调用参数

```python
class MotorNode:
    def __init__(self):
        super().__init__("motor_node")

        # 声明参数，并给默认值
        self.declare_parameter("motor_id", 1)
        self.declare_parameter("max_speed", 6000)
        self.declare_parameter("kp", 1.2)
        self.declare_parameter("ki", 0.1)

        # 读取参数
        self.motor_id = self.get_parameter("motor_id").value
        self.max_speed = self.get_parameter("max_speed").value
        self.kp = self.get_parameter("kp").value
        self.ki = self.get_parameter("ki").value

        self.motor = MotorDriver(self.motor_id)
        self.pid = PID(self.kp, self.ki)

    def cmd_callback(self, msg):
        target_speed = msg.data

        # 使用参数限制最大速度
        if target_speed > self.max_speed:
            target_speed = self.max_speed

        self.motor.set_speed(target_speed)
```



# 工具

## TF坐标变换

暂时不学

## 可视化工具（rqt && RViz）

**rqt**

**RViz**

1. `rviz2` 命令打开界面
2. 左侧Displays界面添加可视化 插件 / 话题



