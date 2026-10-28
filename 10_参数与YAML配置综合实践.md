# 第十章：参数与 YAML 配置综合实践

> 环境：Ubuntu 24.04、ROS 2 Jazzy、C++17。
> 前置：第八章自定义接口、第九章 Launch。
> 本章继续使用熟悉的普通函数和 lambda 写法，不引入新的类继承结构。

## 10.1 这章解决什么问题

之前的发布者在 C++ 中直接填写：

```cpp
message.vehicle_name = "car_01";
message.speed = 5.0;
```

如果改源码中的这些值，需要保存、重新构建并重启程序。

本章把车辆名称和发布速度做成 ROS 参数。我们要实现：

1. 程序运行时读取节点参数，填写车辆状态消息。
2. 用命令行查看和修改参数，观察消息变化。
3. 用 YAML 保存启动设置。
4. 用 Launch 加载 YAML，并启动发布者与已有订阅者。

本例只发布模拟数据，不连接真实车辆，也不会控制真实车辆行驶。

## 10.2 与第三章的关系

第三章已经用 `ros2 param` 修改过海龟背景颜色。那时参数由别人编写的节点提供。

现在我们自己编写节点，决定它提供哪些参数，以及参数如何影响输出。

| 层次 | 负责什么 |
| --- | --- |
| C++ | 声明参数、读取参数，决定如何使用参数 |
| 参数值 | 节点当前使用的配置，例如速度 `8.0` |
| YAML | 保存一组参数的配置文件 |
| Launch | 安排启动程序，并把 YAML 交给目标节点 |
| `.msg` | 仍然规定最终消息的字段结构 |

参数属于节点。两个不同节点可以各有一个同名的 `speed` 参数，它们不自动共享同一个值。

## 10.3 参数与消息字段不要混淆

本章恰好把参数叫 `speed`，消息也有 `speed` 字段，但它们是两份不同用途的数据。

```cpp
double speed = node->get_parameter("speed").as_double();
message.speed = speed;
```

第一句读取节点的配置；第二句把值放进本次要发送的消息。

只有因为程序写了这两步，它们才建立联系。给参数起同样的名字，不会自动修改消息。

## 10.4 新包与沿用内容

本章新建 `vehicle_config_demo`，避免覆盖前面的练习。

| 项目 | 名称 |
| --- | --- |
| 新程序包 | `vehicle_config_demo` |
| 新发布者源码 | `src/configurable_status_pub.cpp` |
| 可执行程序 | `configurable_status_pub` |
| 发布者节点 | `/configurable_status_publisher` |
| 消息类型 | `vehicle_interfaces/msg/VehicleStatus` |
| 话题 | `/vehicle/status` |
| 复用订阅者 | `custom_interface_demo` 包里的 `status_sub` |
| 参数文件 | `config/vehicle.yaml` |
| 启动文件 | `launch/vehicle_config.launch.py` |

本章直接启动第八章订阅者的可执行程序，不包含第九章旧 Launch。因此旧 Launch 中的 `/car/status` 重映射不会自动带到本章。

开始前停止旧练习节点，避免同时出现多个发布者。

## 10.5 创建功能包和目录

在 Ubuntu 终端执行：

```bash
source /opt/ros/jazzy/setup.bash
source ~/ros2_ws/install/setup.bash
cd ~/ros2_ws/src
ros2 pkg create vehicle_config_demo --build-type ament_cmake --license Apache-2.0 --dependencies rclcpp vehicle_interfaces
mkdir -p ~/ros2_ws/src/vehicle_config_demo/launch
mkdir -p ~/ros2_ws/src/vehicle_config_demo/config
```

包内 `src` 放 C++ 源码，`launch` 放启动文件，`config` 放配置文件。

## 10.6 完整发布者代码

```bash
nano ~/ros2_ws/src/vehicle_config_demo/src/configurable_status_pub.cpp
```

填写：

```cpp
#include <chrono>
#include <memory>
#include <string>

#include "rclcpp/rclcpp.hpp"
#include "vehicle_interfaces/msg/vehicle_status.hpp"

using VehicleStatus = vehicle_interfaces::msg::VehicleStatus;

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  auto node = std::make_shared<rclcpp::Node>("configurable_status_publisher");

  // 声明两个参数及其默认值。
  node->declare_parameter<std::string>("vehicle_name", "car_01");
  node->declare_parameter<double>("speed", 5.0);

  auto publisher = node->create_publisher<VehicleStatus>(
    "/vehicle/status", 10);

  auto publish_status = [node, publisher]() {
    // 每次发布前重新读取，才能使用运行时修改后的参数。
    std::string vehicle_name = node->get_parameter("vehicle_name").as_string();
    double speed = node->get_parameter("speed").as_double();

    VehicleStatus message;
    message.vehicle_name = vehicle_name;
    message.speed = speed;
    message.stopped = (speed == 0.0);

    publisher->publish(message);
    RCLCPP_INFO(
      node->get_logger(), "Published: name=%s, speed=%.1f m/s, stopped=%s",
      message.vehicle_name.c_str(), message.speed,
      message.stopped ? "true" : "false");
  };

  auto timer = node->create_wall_timer(
    std::chrono::seconds(1), publish_status);
  rclcpp::spin(node);
  rclcpp::shutdown();
  return 0;
}
```

保存：`Ctrl + O` → Enter → `Ctrl + X`。

### 10.6.1 declare_parameter：声明参数

```cpp
node->declare_parameter<double>("speed", 5.0);
```

告诉节点：提供一个名为 `speed` 的浮点参数，未被启动配置覆盖时默认使用 `5.0`。

它不会把所有普通 C++ 变量自动变成参数。参数需要按代码声明和使用。

### 10.6.2 get_parameter：读取当前参数

```cpp
double speed = node->get_parameter("speed").as_double();
```

读取当前参数，并取出浮点数值。本例 `vehicle_name` 是字符串，所以用 `as_string()`。

### 10.6.3 为什么放在定时器回调里读取

如果只在启动时读一次，然后一直使用那个旧变量，后续修改参数不一定能改变业务输出。

本例每秒发布前都重新读取，因此 `ros2 param set` 成功后，通常下次定时器回调就会使用新值。

这不是所有程序天然都有的行为。参数修改成功，只代表参数值更新；程序是否改变行为，要看代码有没有使用新值。

### 10.6.4 stopped 的计算

```cpp
message.stopped = (speed == 0.0);
```

本例发布的是人为配置的模拟速度，以精确的 `0.0` 表示停车，不是对传感器噪声进行停车判断。

发布周期仍固定为一秒。更改 `speed` 不会改变发布频率。

本章没有添加速度范围校验。练习使用有限、非负的速度值；范围约束、参数校验回调可在后续项目中补充。

## 10.7 创建 YAML 参数文件

```bash
nano ~/ros2_ws/src/vehicle_config_demo/config/vehicle.yaml
```

完整内容：

```yaml
/configurable_status_publisher:
  ros__parameters:
    vehicle_name: "car_yaml"
    speed: 8.0
```

这份文件表示：给指定节点设置这两个启动参数。

- 第一行是目标节点的完整名称，不是包名或可执行程序名。
- `ros__parameters` 中间是两个下划线。
- YAML 用空格缩进，本例每层两个空格，不使用 Tab。
- `speed` 写成 `8.0`，让它被识别为浮点数；不要写成整数 `8`。

修改节点名或命名空间后，应同步检查 YAML 的节点匹配名称，否则可能仍然使用默认值。

## 10.8 完整 Launch 文件

```bash
nano ~/ros2_ws/src/vehicle_config_demo/launch/vehicle_config.launch.py
```

完整内容：

```python
import os

from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    package_share = get_package_share_directory('vehicle_config_demo')
    parameter_file = os.path.join(package_share, 'config', 'vehicle.yaml')

    return LaunchDescription([
        Node(
            package='vehicle_config_demo',
            executable='configurable_status_pub',
            name='configurable_status_publisher',
            parameters=[parameter_file],
            output='screen',
        ),
        Node(
            package='custom_interface_demo',
            executable='status_sub',
            name='config_status_receiver',
            output='screen',
        ),
    ])
```

### 新的三处内容

```python
get_package_share_directory('vehicle_config_demo')
```

查找这个包安装后的共享资源目录，不把你的个人主目录写死在启动文件里。

```python
os.path.join(package_share, 'config', 'vehicle.yaml')
```

拼出 YAML 文件路径。

```python
parameters=[parameter_file]
```

把参数文件交给本次启动的发布者节点。配置路径来自安装目录，所以后面需要安装 `config` 文件夹。

同一个 Launch 可以启动不同包的程序。本例订阅者来自第八章的包。

## 10.9 完整 CMakeLists.txt

```bash
nano ~/ros2_ws/src/vehicle_config_demo/CMakeLists.txt
```

用下面内容配置本章新包：

```cmake
cmake_minimum_required(VERSION 3.8)
project(vehicle_config_demo)

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(vehicle_interfaces REQUIRED)

add_executable(configurable_status_pub src/configurable_status_pub.cpp)
target_compile_features(configurable_status_pub PUBLIC cxx_std_17)
ament_target_dependencies(configurable_status_pub rclcpp vehicle_interfaces)

install(TARGETS
  configurable_status_pub
  DESTINATION lib/${PROJECT_NAME}
)

install(DIRECTORY launch config
  DESTINATION share/${PROJECT_NAME}
)

ament_package()
```

它安装一个新可执行程序，以及 `launch`、`config` 两个资源文件夹。

## 10.10 补充 package.xml 运行依赖

```bash
nano ~/ros2_ws/src/vehicle_config_demo/package.xml
```

保留生成的内容，在最后的 `</package>` 前添加：

```xml
<exec_depend>launch</exec_depend>
<exec_depend>launch_ros</exec_depend>
<exec_depend>ros2launch</exec_depend>
<exec_depend>ament_index_python</exec_depend>
<exec_depend>custom_interface_demo</exec_depend>
```

已有行不重复。放在 `<package>` 内、`<export>` 外。

最后一行声明启动文件会运行第八章包里的订阅者；它不是本章 C++ 程序需要链接的库，所以不用给它添加 CMake 的目标链接依赖。

## 10.11 构建

```bash
cd ~/ros2_ws
colcon build --packages-up-to vehicle_config_demo
source ~/ros2_ws/install/setup.bash
```

该选择包含目标包及其工作空间内依赖，通常涉及接口包、已有程序包和本章程序包。只要依赖可用且构建成功即可，不要求死记输出的包数量。

若构建失败，先处理报错，再运行程序。

## 10.12 实验一：不用配置文件，先看默认值

当前终端：

```bash
ros2 run vehicle_config_demo configurable_status_pub
```

应输出名称 `car_01`、速度 `5.0`、停车状态 `false`。

新开检查终端：

```bash
source /opt/ros/jazzy/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 param list /configurable_status_publisher
ros2 param get /configurable_status_publisher speed
ros2 param get /configurable_status_publisher vehicle_name
```

`speed` 应显示浮点类型和值 `5.0`。此外还可能看到 ROS 2 的其他参数，属于正常现象。

## 10.13 实验二：运行时修改

发布者保持运行，在检查终端执行：

```bash
ros2 param set /configurable_status_publisher speed 12.0
ros2 param get /configurable_status_publisher speed
ros2 topic echo --once /vehicle/status
```

预期参数和消息中的速度变为 `12.0`，不需要重新编译或重启。

刚设置后第一条消息仍可能是此前已发布的数据；继续观察下一周期，程序才会读到新值。

再执行：

```bash
ros2 param set /configurable_status_publisher speed 0.0
```

预期随后发布 `speed: 0.0`、`stopped: true`。这是我们的回调计算出来的，不是参数机制自动判断的。

### 与改源码的区别

| 改什么 | 是否需要重新构建 |
| --- | --- |
| C++ 处理逻辑或默认值 | 需要，之后重启程序 |
| 正在运行节点的参数值 | 本例不需要，用 `ros2 param set` |
| 新增或修改 `.msg` 字段结构 | 需要重新生成接口并构建相关程序 |

`ros2 param set` 不会自动修改源码或 YAML；节点退出后，这次运行时改动不会自动成为下次启动的默认设置。

## 10.14 实验三：Launch 加载 YAML

先按 `Ctrl + C` 停止刚才单独运行的发布者，避免重复启动。

```bash
ros2 launch vehicle_config_demo vehicle_config.launch.py
```

这次启动两个节点，发布者与订阅者应显示：

```text
name=car_yaml, speed=8.0 m/s, stopped=false
```

因为 YAML 中匹配节点的启动配置覆盖了 C++ 默认值。

在另一个已加载环境的终端检查：

```bash
ros2 node list
ros2 topic info /vehicle/status
ros2 param get /configurable_status_publisher speed
```

本例节点名分别为 `/configurable_status_publisher`、`/config_status_receiver`；没有其他发布程序运行时，话题有一个发布者、一个订阅者。

## 10.15 实验四：改 YAML 后重新启动

停止 Launch。打开源码目录中的 `config/vehicle.yaml`，改为：

```yaml
/configurable_status_publisher:
  ros__parameters:
    vehicle_name: "car_02"
    speed: 3.0
```

本章使用普通资源安装方式，所以更新源码目录中的 YAML 后执行：

```bash
cd ~/ros2_ws
colcon build --packages-up-to vehicle_config_demo
source ~/ros2_ws/install/setup.bash
ros2 launch vehicle_config_demo vehicle_config.launch.py
```

预期发布和接收 `car_02`、`3.0`。

这里重新构建是为了更新安装目录中的配置副本，不是因为改参数值必须重新编译 C++。直接使用某个外部 YAML 文件启动时，不必先把它安装进包。

修改文件本身不会自动更新已经运行的节点，本例没有文件自动监视功能。

## 10.16 补充：不用 Launch 也能传入启动参数

先停止其他发布者，再执行其中一种方式。

命令行覆盖默认值：

```bash
ros2 run vehicle_config_demo configurable_status_pub --ros-args -p speed:=6.0 -p vehicle_name:=car_cli
```

从源码目录的文件读取启动参数：

```bash
ros2 run vehicle_config_demo configurable_status_pub --ros-args --params-file ~/ros2_ws/src/vehicle_config_demo/config/vehicle.yaml
```

`--ros-args` 后面是 ROS 参数选项；`-p` 指定一个参数赋值，`--params-file` 指定参数文件。

这两种方式只启动发布者，没有替你启动订阅者。

## 10.17 保存当前参数与重新加载

当节点运行时，检查终端执行：

```bash
ros2 param dump /configurable_status_publisher > ~/vehicle_saved.yaml
```

`dump` 原本打印到终端；`>` 把输出写入主目录的文件。该文件已存在时会覆盖它，所以本例使用单独的保存文件名。

查看：

```bash
cat ~/vehicle_saved.yaml
```

文件可能还包含 ROS 2 自动提供的其他参数，不要求只出现两个自定义参数。

对正在运行的同一节点重新加载：

```bash
ros2 param load /configurable_status_publisher ~/vehicle_saved.yaml
```

关注每项参数是否设置成功。某些系统参数可能只读，不能假定所有导出项都可在运行时重新设置。

保存文件并不代表下次会自动加载它；需要在启动时明确指定文件，或由 Launch 引用。

## 10.18 参数、重映射、消息字段的比较

| 操作 | 本例效果 | 不会自动做的事 |
| --- | --- | --- |
| 参数 `speed=8.0` | 经回调读取后，影响发送的速度值 | 不改变话题名称或消息字段结构 |
| 重映射 `/vehicle/status` 到 `/car/status` | 改变运行时通信名称 | 不改速度数值或消息类型 |
| 修改 `VehicleStatus.msg` | 改变消息类型结构 | 不自动实现新字段的数据获取逻辑 |
| Launch 的 `name` | 改变运行时节点名称 | 不修改可执行程序名 |

不要把 Launch 的 `name='...'` 与参数 `vehicle_name` 混淆：前者是 ROS 节点名，后者是本例放进消息里的车辆名称。

## 10.19 常见问题

| 现象 | 优先检查 |
| --- | --- |
| YAML 没有生效，仍是默认值 | 节点名称匹配、`ros__parameters`、文件路径、安装副本是否更新 |
| 修改速度提示参数类型不匹配 | 浮点参数写 `8.0`，不要写整数 `8` |
| 参数修改成功，但旧程序输出不变 | 是否运行本章新程序；代码是否每次读取当前值 |
| 两种速度交替出现 | 是否有旧发布者仍在同一话题发送 |
| 消息速度变了，频率没变 | 正常，速度参数不控制本例定时器周期 |
| 重启后参数恢复 | 运行时修改未保存，或启动时没有加载保存文件 |
| 找不到配置文件 | `config` 安装规则、文件名、构建和环境加载是否正确 |
| 订阅者收不到数据 | 是否误用了第九章仍重映射到 `/car/status` 的旧 Launch |

## 10.20 命令速查

| 用途 | 命令 |
| --- | --- |
| 查看参数列表 | `ros2 param list /configurable_status_publisher` |
| 读取速度 | `ros2 param get /configurable_status_publisher speed` |
| 运行时设置速度 | `ros2 param set /configurable_status_publisher speed 12.0` |
| 保存参数 | `ros2 param dump /configurable_status_publisher > ~/vehicle_saved.yaml` |
| 运行时加载文件 | `ros2 param load /configurable_status_publisher ~/vehicle_saved.yaml` |
| 默认值启动 | `ros2 run vehicle_config_demo configurable_status_pub` |
| 配置及双节点启动 | `ros2 launch vehicle_config_demo vehicle_config.launch.py` |

## 10.21 自检题与答案

1. 在 C++ 中声明参数的函数是什么？读取参数的函数是什么？
2. 参数 `speed` 与消息字段 `speed` 会因为同名就自动同步吗？
3. 为什么参数要在发布回调中重新读取？
4. `ros2 param set` 会直接修改 YAML 文件吗？
5. YAML 第一行应该写包名、可执行程序名还是节点名称？
6. 为什么修改源码目录中的 YAML 后，本章还执行一次构建？
7. 把速度设成零后，`stopped` 为真是谁决定的？

答案：

1. `declare_parameter` 声明，`get_parameter` 读取。
2. 不会，需要代码读取参数并填写消息。
3. 为了让运行时更新后的参数参与后续发布，而不一直使用启动时缓存的旧值。
4. 不会，它修改正在运行的节点中的参数值。
5. 本例写目标节点的完整名称。
6. 将更新的资源副本安装到 Launch 实际读取的位置，不是重新实现业务逻辑。
7. 回调里的 `message.stopped = (speed == 0.0);`。

## 10.22 完成本章的标准

- [ ] 能声明并读取自己节点的参数。
- [ ] 能运行时修改速度，验证实际消息发生变化。
- [ ] 能解释参数更新与程序行为变化之间的代码联系。
- [ ] 能用 YAML 设置启动值。
- [ ] 能用 Launch 同时加载参数和启动两个节点。
- [ ] 能区分默认值、启动配置和本次运行时修改。
- [ ] 能区分节点名称、车辆名称、话题名称。

## 10.23 十章基础课程结束后

完成这十章，表示第一轮基础入门完成，不代表 ROS 2 所有内容都学完，也不要求独立默写全部代码。

你已经接触并实践：节点与功能包、话题、服务、动作、参数、C++ 节点、构建、自定义接口、Launch 与配置文件。

下一步适合做一个小型综合项目，练习自己拆分节点、选择通信方式和排查问题，再按项目需要学习 TF2、URDF、RViz、仿真、数据记录或导航。动作的 C++ 实现、QoS 和更完整的参数校验也可以继续深入。

## 参考资料与验证范围

- [ROS 2 Jazzy：C++ 节点中的参数](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Using-Parameters-In-A-Class-CPP.html)
- [ROS 2 Jazzy：理解参数](https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Parameters/Understanding-ROS2-Parameters.html)
- [ROS 2 Jazzy：Launch 管理项目和加载参数文件](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Launch/Using-ROS2-Launch-For-Large-Projects.html)
- 上述文档的预览站地址可将域名 `docs.ros.org` 换成 `repo.test.ros2.org`。

已核对 Jazzy 参数及配置用法，教学代码按当前课程进度设计；本次未在你的 Ubuntu 虚拟机中构建或运行，实际验证会在逐步实操中完成。
