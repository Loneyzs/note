# Kuavo ROS2 仿真控制快速上手

适用状态：你已经能在 Windows + WSL 中进入 `kuavo-ros-opensource` 的 Docker 容器，并且能开始编译 ROS1 仿真环境。

本文目标：不改动底层仓库，先用 ROS1 + MuJoCo 跑起仿真，再通过 ROS1-ROS2 bridge 用 ROS2 写几个最小脚本控制机器人。

---

## 1. 先理解整体结构

这个仓库主体不是 ROS2 工程，而是：

```text
kuavo-ros-opensource
  = ROS1 Noetic
  + catkin
  + roslaunch
  + MuJoCo / Gazebo 仿真
```

ROS2 的使用方式是外接一个桥接工程：

```text
ROS2 脚本 / ROS2 节点
        |
        v
ros1_bridge
        |
        v
ROS1 Noetic 控制与仿真节点
        |
        v
MuJoCo 仿真机器人
```

所以不要一开始就尝试把 `kuavo-ros-opensource` 改成 ROS2。正确入门路径是：

1. 用 ROS1 启动 MuJoCo 仿真。
2. 用 `kuavo_ros2_to_ros1_bridge` 启动 ROS1-ROS2 桥。
3. 在 ROS2 侧写脚本，发布 `/cmd_vel`、`/cmd_pose` 等控制话题。

---

## 2. 三个终端的分工

后续建议固定使用 3 个终端。

### 终端 1：启动 ROS1 仿真

进入 WSL：

```powershell
wsl -d Ubuntu24
```

进入仓库：

```bash
cd /mnt/e/Files/ros2_projects/kuavo-ros-opensource
./docker/run.sh
```

进入容器后，通常在：

```bash
/root/kuavo_ws
```

如果还没编译，先编译：

```bash
rm -rf devel build
catkin config -DCMAKE_ASM_COMPILER=/usr/bin/as -DCMAKE_BUILD_TYPE=Release
source installed/setup.zsh
catkin build humanoid_controllers
```

启动 MuJoCo 仿真：

```bash
source devel/setup.zsh
roslaunch humanoid_controllers load_kuavo_mujoco_sim.launch
```

这个终端保持运行，不要关闭。

---

### 终端 2：启动 ROS1-ROS2 bridge

官方文档要求使用另一个仓库：

```bash
git clone https://gitee.com/leju-robot/kuavo_ros2_to_ros1_bridge.git
```

进入 bridge 仓库：

```bash
cd ~/kuavo_ros2_to_ros1_bridge
sudo ./docker/run_bridge.sh
```

第一次使用需要构建消息和桥：

```bash
./build_msg_and_bridge.sh
```

启动全部 topic 桥接：

```bash
source bridge_ws/install/setup.bash
ros2 run ros1_bridge dynamic_bridge --bridge-all-topics
```

这个终端也保持运行，不要关闭。

---

### 终端 3：运行 ROS2 脚本

再开一个 bridge 容器终端：

```bash
cd ~/kuavo_ros2_to_ros1_bridge
sudo ./docker/run_bridge.sh
```

加载 ROS2 环境：

```bash
source /opt/ros/foxy/setup.bash
source bridge_ws/install/setup.bash
```

检查 ROS2 能否看到 ROS1 仿真的话题：

```bash
ros2 topic list
```

重点看有没有：

```text
/cmd_vel
/cmd_pose
/cmd_pose_world
/kuavo_arm_traj
```

查看话题类型：

```bash
ros2 topic info /cmd_vel
ros2 topic info /cmd_pose
```

通常 `/cmd_vel` 和 `/cmd_pose` 都是：

```text
geometry_msgs/msg/Twist
```

---

## 3. 不写代码，先用 ROS2 命令测试

先发一个很小的前进速度，持续 1 次：

```bash
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.1, y: 0.0, z: 0.0}, angular: {z: 0.0}}"
```

停止：

```bash
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.0, y: 0.0, z: 0.0}, angular: {z: 0.0}}"
```

如果桥接和仿真都正常，机器人应收到 ROS2 侧发出的控制指令。

注意：速度不要一开始设太大。建议先从 `0.05` 到 `0.1` 开始。

---

## 4. 脚本 1：ROS2 发布速度控制

在终端 3 中创建一个练习目录：

```bash
mkdir -p ~/kuavo_ros2_play
cd ~/kuavo_ros2_play
```

创建 `walk_forward.py`：

```python
#!/usr/bin/env python3
import time

import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist


class WalkForward(Node):
    def __init__(self):
        super().__init__("walk_forward")
        self.pub = self.create_publisher(Twist, "/cmd_vel", 10)

    def publish_cmd(self, vx: float, vy: float, wz: float):
        msg = Twist()
        msg.linear.x = vx
        msg.linear.y = vy
        msg.linear.z = 0.0
        msg.angular.z = wz
        self.pub.publish(msg)


def main():
    rclpy.init()
    node = WalkForward()

    try:
        # 给 publisher 一点发现订阅者的时间
        time.sleep(1.0)

        # 慢速前进 2 秒
        start = time.time()
        while time.time() - start < 2.0:
            node.publish_cmd(0.1, 0.0, 0.0)
            rclpy.spin_once(node, timeout_sec=0.05)
            time.sleep(0.05)

        # 停止
        for _ in range(10):
            node.publish_cmd(0.0, 0.0, 0.0)
            rclpy.spin_once(node, timeout_sec=0.05)
            time.sleep(0.05)

    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == "__main__":
    main()
```

运行：

```bash
source /opt/ros/foxy/setup.bash
source bridge_ws/install/setup.bash
python3 walk_forward.py
```

现象：机器人慢速前进约 2 秒，然后停止。

---

## 5. 脚本 2：原地缓慢转向

创建 `turn_left.py`：

```python
#!/usr/bin/env python3
import time

import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist


class TurnLeft(Node):
    def __init__(self):
        super().__init__("turn_left")
        self.pub = self.create_publisher(Twist, "/cmd_vel", 10)

    def publish_cmd(self, wz: float):
        msg = Twist()
        msg.linear.x = 0.0
        msg.linear.y = 0.0
        msg.linear.z = 0.0
        msg.angular.z = wz
        self.pub.publish(msg)


def main():
    rclpy.init()
    node = TurnLeft()

    try:
        time.sleep(1.0)

        # 原地左转 2 秒。角速度单位是 rad/s。
        start = time.time()
        while time.time() - start < 2.0:
            node.publish_cmd(0.2)
            rclpy.spin_once(node, timeout_sec=0.05)
            time.sleep(0.05)

        # 停止
        for _ in range(10):
            node.publish_cmd(0.0)
            rclpy.spin_once(node, timeout_sec=0.05)
            time.sleep(0.05)

    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == "__main__":
    main()
```

运行：

```bash
python3 turn_left.py
```

---

## 6. 脚本 3：位置增量控制

`/cmd_pose` 通常也是 `geometry_msgs/msg/Twist`，但语义和 `/cmd_vel` 不同：

- `/cmd_vel`：速度控制。
- `/cmd_pose`：基于当前位置的位姿增量控制。

创建 `move_pose_once.py`：

```python
#!/usr/bin/env python3
import time

import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist


class MovePoseOnce(Node):
    def __init__(self):
        super().__init__("move_pose_once")
        self.pub = self.create_publisher(Twist, "/cmd_pose", 10)

    def send_pose_delta(self):
        msg = Twist()
        msg.linear.x = 0.2   # 相对当前位置向前 0.2 m
        msg.linear.y = 0.0
        msg.linear.z = 0.0
        msg.angular.z = 0.0  # yaw 增量，单位 rad
        self.pub.publish(msg)


def main():
    rclpy.init()
    node = MovePoseOnce()

    try:
        time.sleep(1.0)
        node.send_pose_delta()
        rclpy.spin_once(node, timeout_sec=0.5)
        print("已发布 /cmd_pose 位姿增量")
    finally:
        node.destroy_node()
        rclpy.shutdown()


if __name__ == "__main__":
    main()
```

运行：

```bash
python3 move_pose_once.py
```

注意：不要同时持续发布 `/cmd_vel` 和 `/cmd_pose`。文档里明确提到位姿指令优先级较高，混用容易让行为难以判断。

---

## 7. 官方 ROS2 示例

如果你已经按官方 `kuavo_ros2_to_ros1_bridge` 文档编译了示例，可以直接运行：

```bash
source /opt/ros/foxy/setup.bash
source ros2/install/setup.bash
source kuavo_example/install/setup.bash
```

速度控制：

```bash
ros2 run kuavo_example_py cmd_vel
```

位置控制：

```bash
ros2 run kuavo_example_py cmd_pose
```

世界坐标位置控制：

```bash
ros2 run kuavo_example_py cmd_pose_world
```

单步控制：

```bash
ros2 run kuavo_example_py single_step_control
```

手臂轨迹控制：

```bash
ros2 run kuavo_example_py arm_traj_control
```

头部控制：

```bash
ros2 run kuavo_example_py head_control
```

初学阶段建议先跑官方示例，再自己写最小脚本。

---

## 8. 常见问题排查

### 1. ROS2 看不到 `/cmd_vel`

检查终端 2 的 bridge 是否还在运行：

```bash
ros2 run ros1_bridge dynamic_bridge --bridge-all-topics
```

检查终端 1 的 ROS1 仿真是否还在运行。

### 2. 发布了 `/cmd_vel`，机器人没动

先确认 topic 类型：

```bash
ros2 topic info /cmd_vel
```

再确认发布命令是否真的在发：

```bash
ros2 topic echo /cmd_vel
```

如果 echo 能看到，但机器人不动，可能是仿真节点还没完全启动，或者当前控制模式不接收该指令。

### 3. bridge 报 ROS master 连接失败

ROS1 仿真端必须先启动。顺序建议是：

```text
先启动 ROS1 MuJoCo 仿真
再启动 bridge
最后运行 ROS2 脚本
```

### 4. 不确定某个话题应该怎么发

先查类型：

```bash
ros2 topic info /话题名
```

再查消息结构：

```bash
ros2 interface show geometry_msgs/msg/Twist
ros2 interface show sensor_msgs/msg/JointState
```

---

## 9. 最小学习路线

建议按这个顺序学：

1. Docker 只先学 4 个概念：镜像、容器、挂载、进入容器。
2. ROS1 只先学：`roslaunch`、`rostopic list`、`rostopic echo`。
3. ROS2 只先学：`ros2 topic list`、`ros2 topic pub`、`rclpy` 发布者。
4. 先用 `/cmd_vel` 做速度控制。
5. 再用 `/cmd_pose` 做位姿控制。
6. 最后再碰手臂、灵巧手、头部和自定义消息。

---

## 10. 当前阶段不要做的事

暂时不要做这些：

- 不要把 `kuavo-ros-opensource` 直接迁移成 ROS2。
- 不要一开始就改 MPC/WBC 控制器。
- 不要同时混发 `/cmd_vel` 和 `/cmd_pose`。
- 不要一上来发很大的速度或角速度。
- 不要在 Windows PowerShell 里直接运行 Linux `.sh` 脚本。

最稳路径是：

```text
ROS1 仿真跑通
  -> bridge 跑通
  -> ROS2 能看到 topic
  -> ROS2 发布 /cmd_vel
  -> ROS2 写自己的 rclpy 小脚本
```

