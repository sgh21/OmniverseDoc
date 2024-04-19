### 机器人控制和仿真平台
#### 库概述
+ omni.isaac.core
  + 基本机器人的定义和操作
  + 内部类
    + Articulations:关节控制 Robots:关节集合的控制和信息读取
    + logger:数据存储和读取
    + Objects:已经实现的几何/物理几何体
    + World/Simulation contex:好像两者是等价的，都是仿真控制程序，接口和参数都类似
    + Scene:stage实例添加，清除，对象的获取等
+ omni.isaac.dynamic_control
  + 机器人动力学控制
+ omni.isaac.utils
  + 数学运算库
+ omni.isaac.importer
  + 机器人描述文件支持 URDF、MCJF
+ isaac Replicator
  + 3D数据合成
+ Lula
  + 底层运动控制，逆运动学求解器，运动生成等
#### 开发流程
+ 开发工作流
  + GUI、Extension、独立Python脚本、jupyter
  + 支持和ROS互通，进行控制算法的验证
+ 仿真数据格式
  + RGB、深度点云、鱼眼
#### GUI
![alt text](image.png)
+ isaac utils
  + 机器人开发的文件导入等
+ isaac examples
  + 参考案例
+ synthetic data
  + 数据合成相关包
+ windows\extension
  + 拓展包
#### 基于Extension的开发
+ 通过Python设计extension，实现extension和仿真热交互
+ 方便导入已存在的extension，并修改源码  

![alt text](image-1.png)

#### 基于Srcipts Editer
![alt text](image-2.png)
#### 完全基于Python
+ 在安装目录上有独立python_example
![alt text](image-3.png)
### 开发案例
#### GUI创建机器人和控制
+ 教程链接：https://www.bilibili.com/video/BV1Ee4y1m71J/?spm_id_from=333.337.search-card.all.click&vd_source=55d021e2a2b2b82298ad677593d59ac3
+ 2022版本后，单位为m
+ 步骤概述
  + 创建实体
  + 添加物理碰撞属性
  + 添加关节
  + 添加关节驱动器
+ 机器人文件导入
  + URDF、MJCF(mujoco):Isaac Utils->workflows->MJCF importer/URDF importer
  + 检查文件是否正确：增益调节器和关节检查器
#### 自定义机器人URDF文件导入  

![alt text](28bc41c4a686a6ad068da9915a3dff1.png)

+ GUI导入和调试  

  ![alt text](image-4.png)
  + importer:导入URDF/MJCF
  + Articulation Inspector:机器人关节查看
  + Gain Tuner:关节PID参数调节(重力和增益置零)
+ 基于Hello World示例
  + Isaac example\Hello World
  + 引入omni.isaac.core.utils.stage import add_reference_to_stage
  + 在源码中添加usd文件路径和要导入Stage下的路径
  + add_reference_to_stage(usd_path=usd_path,prim_path=prim_path)
+ 导入传感器
  + create->Isaac->Sensors->...
  + 将相机放在哪个Xform下，它就会跟随哪个Xform移动
#### 自定义控制器实现
+ Omniverse Isaac Sim 中的控制器继承自 BaseController 接口。您将需要实现一个forward方法，并且它必须返回一个ArticulationAction类型
+ 示例代码
```python
from omni.isaac.examples.base_sample import BaseSample
from omni.isaac.core.utils.nucleus import get_assets_root_path
from omni.isaac.wheeled_robots.robots import WheeledRobot
from omni.isaac.core.utils.types import ArticulationAction
from omni.isaac.core.controllers import BaseController
import numpy as np


class CoolController(BaseController):
    def __init__(self):
        super().__init__(name="my_cool_controller")
        # An open loop controller that uses a unicycle model
        self._wheel_radius = 0.03
        self._wheel_base = 0.1125
        return

    def forward(self, command):
        # command will have two elements, first element is the forward velocity
        # second element is the angular velocity (yaw only).
        joint_velocities = [0.0, 0.0]
        joint_velocities[0] = ((2 * command[0]) - (command[1] * self._wheel_base)) / (2 * self._wheel_radius)
        joint_velocities[1] = ((2 * command[0]) + (command[1] * self._wheel_base)) / (2 * self._wheel_radius)
        # A controller has to return an ArticulationAction
        return ArticulationAction(joint_velocities=joint_velocities)


class HelloWorld(BaseSample):
    def __init__(self) -> None:
        super().__init__()
        return

    def setup_scene(self):
        world = self.get_world()
        world.scene.add_default_ground_plane()
        assets_root_path = get_assets_root_path()
        jetbot_asset_path = assets_root_path + "/Isaac/Robots/Jetbot/jetbot.usd"
        world.scene.add(
            WheeledRobot(
                prim_path="/World/Fancy_Robot",
                name="fancy_robot",
                wheel_dof_names=["left_wheel_joint", "right_wheel_joint"],
                create_robot=True,
                usd_path=jetbot_asset_path,
            )
        )
        return

    async def setup_post_load(self):
        self._world = self.get_world()
        self._jetbot = self._world.scene.get_object("fancy_robot")
        self._world.add_physics_callback("sending_actions", callback_fn=self.send_robot_actions)
        # Initialize our controller after load and the first reset
        self._my_controller = CoolController()
        return

    def send_robot_actions(self, step_size):
        #apply the actions calculated by the controller
        self._jetbot.apply_action(self._my_controller.forward(command=[0.20, np.pi / 4]))
        return
```
### Core API
+ 从USD Python 和Pixar的接口中抽象出的专门用于机器人应用程序的API
+ World 核心类，提供了关于时间的相关时间，添加回调函数，迭代物理时间步，重置场景，添加任务
  + Scene实例，管理USD stage的仿真资产，实现stage中添加，操作和检查等
  + World只能有一个实例
  + 通过setup_scene()方法，从空的stage中添加资产
+ Task 类，提供了另一种场景创建、信息检索和计算指标模块儿化的方法，使用高级逻辑创造更加复杂的场景
  + 通过setup_scene()方法添加3D资产到场景，通过get_observation()从simulator获取观测信息，通过pre_step()来逐步执行控制指令。
### api阅读
+ async关键字用于定义一个异步函数，表明该函数是一个协程，可以在其中使用await关键字等待其他异步操作完成
异步函数的执行不会阻塞事件循环，而是会立即返回一个协程对象
+ omni.isaac.core
  + 基本机器人的定义和操作
  + 内部类
    + Articulations:关节控制 Robots:关节集合的控制和信息读取
    + logger:数据存储和读取
    + Objects:已经实现的几何/物理几何体
    + World/Simulation contex:好像两者是等价的，都是仿真控制程序，接口和参数都类似
    + Scene:stage实例添加，清除，对象的获取等
+ omni.isaac.dynamic_control
  + 机器人动力学控制
+ omni.isaac.utils
  + 数学运算库
+ omni.isaac.importer
  + 机器人描述文件支持 URDF、MCJF
+ isaac Replicator
  + 3D数据合成
+ Lula
  + 底层运动控制，逆运动学求解器，运动生成等
### 教程和资料
+ 官方教程
  + https://www.youtube.com/watch?v=nleDq-oJjGk
  + https://www.youtube.com/watch?v=1RSugmJ4_gs
  + https://www.youtube.com/watch?v=nXM5_mwUFOI
  + https://www.youtube.com/watch?v=VrTVUpDM7K8
  + https://www.youtube.com/watch?v=RhjRrUK2abs
  + https://www.youtube.com/watch?v=i4fGVc6lImo softbody simulation
  + https://www.youtube.com/watch?v=WhaybakLTXE
+ isaac_sim app
  + https://www.youtube.com/playlist?list=PL3jK4xNnlCVf1SzxjCm7ZxDBNl9QYyV8X
+ isaac_sim robots:好像是关于学习和机器人控制的，可以看看
  + https://www.youtube.com/playlist?list=PL3jK4xNnlCVdi9t6KQq6Z-cwhiVXlJv2p
+ b站公开课
  + https://www.bilibili.com/video/BV1Ee4y1m71J/?spm_id_from=333.337.search-card.all.click&vd_source=55d021e2a2b2b82298ad677593d59ac3
+ 知乎demo
  + https://zhuanlan.zhihu.com/p/546744364
+ 官网demo
  + https://docs.omniverse.nvidia.com/isaacsim/latest/how_to_guides/robots_simulation.html
+ hello robot
  + https://docs.omniverse.nvidia.com/isaacsim/latest/core_api_tutorials/tutorial_core_hello_robot.html
+ 机器人相关库
  + 关节控制：[omni.isaac.dynamic_control](https://docs.omniverse.nvidia.com/py/isaacsim/source/extensions/omni.isaac.dynamic_control/docs/index.html)
  + 运动生成和逆运动学：https://docs.omniverse.nvidia.com/isaacsim/latest/concepts/motion_generation/index.html#