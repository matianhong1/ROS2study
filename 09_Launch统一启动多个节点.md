# 第九章：Launch——统一启动多个节点

> 环境：Ubuntu 24.04、ROS 2 Jazzy。
> 前置：第八章的 `vehicle_interfaces` 和 `custom_interface_demo` 已构建并运行成功。
> 学习方式：先理解文件结构，再逐步操作。Python 启动文件不要求提前系统学习 Python。

## 9.1 本章目标

第八章运行发布者和订阅者，需要在两个终端分别执行：

```bash
ros2 run custom_interface_demo status_pub
ros2 run custom_interface_demo status_sub
```

本章把它们整理到一个启动文件中，之后执行：

```bash
ros2 launch custom_interface_demo vehicle_status.launch.py
```

就能启动两个程序，并在一个终端观察日志、发出统一停止请求。

本章要学会：

- 看懂一个基本的 Python Launch 文件。
- 区分包名、可执行程序名、节点名、启动文件名。
- 在 ament_cmake 包中安装 Launch 文件。
- 用 `ros2 launch` 启动和停止多个节点。
- 不改 C++ 源码，配置节点名称及话题重映射。
- 解释为什么启动列表的先后顺序不等于服务已经准备就绪。

## 9.2 Launch 是什么

Launch 可以理解为“启动安排”：规定运行哪些程序，以及运行时给它们哪些配置。

本章的职责划分：

| 文件 | 负责什么 |
| --- | --- |
| `.msg`、`.srv` | 定义通信数据结构 |
| `.cpp` | 实现节点业务，例如发送状态、执行求和 |
| `CMakeLists.txt` | 配置构建和安装 |
| `.launch.py` | 描述启动哪些程序及运行配置 |

Launch 文件不会替你编译 C++，也不会代替 `publish`、订阅回调、服务回调。原来两个节点的通信逻辑仍在原程序中。

本章采用 Python Launch。ROS 2 也支持 XML、YAML 等形式，不要求三种一起学习。

## 9.3 本章沿用的项目

继续使用第八章的程序包 `custom_interface_demo`，不新建包，也不改接口包。

| 项目 | 名称 |
| --- | --- |
| 程序包 | `custom_interface_demo` |
| 发布者可执行程序 | `status_pub` |
| 订阅者可执行程序 | `status_sub` |
| 原发布节点名 | `/status_publisher` |
| 原订阅节点名 | `/status_subscriber` |
| 原话题 | `/vehicle/status` |
| 消息类型 | `vehicle_interfaces/msg/VehicleStatus` |
| 新启动文件 | `launch/vehicle_status.launch.py` |

先保证之前手动启动的练习程序已按 `Ctrl + C` 停止，避免重复启动导致节点重名、发布者数量增加或日志混淆。

## 9.4 准备与创建文件

启动 Ubuntu，按 `Ctrl + Alt + T` 打开终端：

```bash
source /opt/ros/jazzy/setup.bash
source ~/ros2_ws/install/setup.bash
mkdir -p ~/ros2_ws/src/custom_interface_demo/launch
nano ~/ros2_ws/src/custom_interface_demo/launch/vehicle_status.launch.py
```

`launch` 是程序包内新建的文件夹，与该包的 `src` 并列。

这次文件后缀是 `.launch.py`，不是 `.cpp`。Linux 文件名区分大小写。

## 9.5 第一个完整启动文件

在 `vehicle_status.launch.py` 中填写：

```python
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    publisher = Node(
        package='custom_interface_demo',
        executable='status_pub',
        output='screen',
    )

    subscriber = Node(
        package='custom_interface_demo',
        executable='status_sub',
        output='screen',
    )

    return LaunchDescription([
        publisher,
        subscriber,
    ])
```

保存：`Ctrl + O` → Enter → `Ctrl + X`。

### 9.5.1 两句 import

```python
from launch import LaunchDescription
from launch_ros.actions import Node
```

把要用的 Python 类型导入当前文件。用途可以类比 C++ 引入声明，但不是相同的语言机制。

- `LaunchDescription`：保存启动动作的描述。
- 这里的 `Node`：描述“启动一个 ROS 节点程序”的动作。

此处的 `Node(...)` 不是 C++ 的 `std::make_shared<rclcpp::Node>(...)`。前者安排运行可执行程序；程序启动后，C++ 代码才创建实际节点。

### 9.5.2 函数入口

```python
def generate_launch_description():
```

`def` 定义 Python 函数。ROS 2 加载这个文件时，会调用这个函数取得启动描述，因此名称要写对。

Python 用缩进表示代码属于哪个部分，不使用 C++ 的大括号。本例统一使用四个空格缩进，不混用 Tab。

### 9.5.3 Node 中的配置

```python
publisher = Node(
    package='custom_interface_demo',
    executable='status_pub',
    output='screen',
)
```

| 配置 | 含义 |
| --- | --- |
| `package` | 可执行程序属于哪个包 |
| `executable` | 要启动哪个已构建、安装的程序 |
| `output='screen'` | 将进程输出显示在启动终端中 |

这里填 `status_pub`，不是 `status_pub.cpp`，也不是节点名称 `status_publisher`。

`publisher` 是 Python 变量名，保存启动动作；它不是 C++ 的发布者接口对象。

### 9.5.4 返回启动描述

```python
return LaunchDescription([
    publisher,
    subscriber,
])
```

把两个启动动作交给 Launch 系统执行。

列表中的顺序不保证第一个节点已经完成初始化，第二个才开始运行。这里的两个进程可以并发运行。

## 9.6 安装 Launch 文件

目前文件在源码目录中。要通过“包名 + 文件名”查找启动文件，还需要把它安装到包的共享目录。

打开：

```bash
nano ~/ros2_ws/src/custom_interface_demo/CMakeLists.txt
```

保留第八章的全部程序配置。在最后的 `ament_package()` 前新增：

```cmake
install(DIRECTORY launch
  DESTINATION share/${PROJECT_NAME}
)
```

这一段会把整个 `launch` 文件夹安装到包的共享目录下，保留该文件夹名称。

与之前安装可执行程序的配置比较：

| 安装内容 | 典型目的地 |
| --- | --- |
| `install(TARGETS ...)`：程序 | `lib/${PROJECT_NAME}` |
| `install(DIRECTORY launch ...)`：启动文件夹 | `share/${PROJECT_NAME}` |

新增的这段不替换已有的 `install(TARGETS ...)`。`ament_package()` 仍在最后。

本章不需要给 `.launch.py` 添加 `add_executable`，也不需要用 `g++` 编译它。

## 9.7 声明运行依赖

打开同一个程序包的 `package.xml`：

```bash
nano ~/ros2_ws/src/custom_interface_demo/package.xml
```

保留已有内容，在最后的 `</package>` 前添加：

```xml
<exec_depend>launch</exec_depend>
<exec_depend>launch_ros</exec_depend>
<exec_depend>ros2launch</exec_depend>
```

若某行已有，不重复添加。三行都位于 `<package>` 内、`<export>` 外。

它们声明运行启动文件及启动命令需要的依赖，不会修改 C++ 消息字段或通信逻辑。

若运行时确认缺少这些软件包，再安装：

```bash
sudo apt install ros-jazzy-launch ros-jazzy-launch-ros ros-jazzy-ros2launch
```

通常桌面安装已有，不需要提前重复安装。

## 9.8 构建并一键启动

```bash
cd ~/ros2_ws
colcon build --packages-up-to custom_interface_demo
source ~/ros2_ws/install/setup.bash
ros2 launch custom_interface_demo vehicle_status.launch.py
```

命令通式：

```bash
ros2 launch 包名 启动文件名
```

构建这一步会执行安装规则，让 Launch 文件出现在安装目录。虽然没有修改 C++，新增的启动文件也需要安装后才能稳定地按包名查找。

预期终端出现两个进程启动信息，随后交替看到：

```text
[status_publisher]: Published: name=car_01, speed=5.0 m/s, stopped=false
[status_subscriber]: Received: name=car_01, speed=5.0 m/s, stopped=false
```

时间戳、进程标签、输出顺序可以不同。两个程序输出到同一终端，所以日志交错是正常现象。

若你上一章改过发布数据，显示值以当前 C++ 程序为准；Launch 不会把速度自动恢复成 `5.0`。

## 9.9 检查节点和通信

保留 Launch 运行，新开检查终端：

```bash
source /opt/ros/jazzy/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 node list
ros2 topic info /vehicle/status
ros2 topic echo --once /vehicle/status
```

没有其他练习程序运行时，预期能看到 `/status_publisher` 和 `/status_subscriber`，话题类型仍为 `vehicle_interfaces/msg/VehicleStatus`。

Launch 改变了启动方式，没有把消息类型换掉，也没有把两个程序合并成一个节点。

可以额外运行 `rqt_graph` 查看通信关系，但本章不要求必须使用它。

## 9.10 统一停止

回到运行 `ros2 launch` 的终端，按：

```text
Ctrl + C
```

Launch 会向它管理的进程发出停止信号。本章程序使用正常的 ROS 2 退出处理，通常会随之退出。

它不会替你停止之前在其他终端单独启动的节点。

稍等后在检查终端执行 `ros2 node list`，确认本次启动的节点已退出。发现旧结果时可短暂等待发现信息更新，不要据此直接认为退出失败。

## 9.11 实验一：配置节点名称

停止上一个 Launch 后，把 `vehicle_status.launch.py` 改成下面完整内容：

```python
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    return LaunchDescription([
        Node(
            package='custom_interface_demo',
            executable='status_pub',
            name='vehicle_sender',
            output='screen',
        ),
        Node(
            package='custom_interface_demo',
            executable='status_sub',
            name='vehicle_receiver',
            output='screen',
        ),
    ])
```

构建、加载环境、重新启动：

```bash
cd ~/ros2_ws
colcon build --packages-up-to custom_interface_demo
source ~/ros2_ws/install/setup.bash
ros2 launch custom_interface_demo vehicle_status.launch.py
```

检查 `ros2 node list`，预期节点名变为 `/vehicle_sender` 和 `/vehicle_receiver`。

可执行程序仍叫 `status_pub`、`status_sub`，C++ 文件也没有改名；话题仍是 `/vehicle/status`。

这里由启动配置通过 ROS 参数覆盖节点名称。我们的 C++ 程序已经通过 `rclcpp::init(argc, argv)` 接收这些参数。

## 9.12 实验二：话题重映射

重映射是运行时把一个通信名称换成另一个名称，不改变消息类型或字段。

本例两个 C++ 程序都使用 `/vehicle/status`。如果想让它们改在 `/car/status` 上通信，可以在双方启动动作中都设置重映射。

停止旧 Launch，将文件改成：

```python
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    return LaunchDescription([
        Node(
            package='custom_interface_demo',
            executable='status_pub',
            name='vehicle_sender',
            remappings=[('/vehicle/status', '/car/status')],
            output='screen',
        ),
        Node(
            package='custom_interface_demo',
            executable='status_sub',
            name='vehicle_receiver',
            remappings=[('/vehicle/status', '/car/status')],
            output='screen',
        ),
    ])
```

重新构建、加载环境并启动。在检查终端执行：

```bash
ros2 topic list -t
ros2 topic info /car/status
ros2 topic echo --once /car/status
```

预期话题名称变为 `/car/status`，类型仍是 `vehicle_interfaces/msg/VehicleStatus`。

如果只给发布者改名，订阅者仍订阅旧话题，两者就无法通过原来的对应关系通信。双方都改到同一个新名称，才能继续通信。

这里的 `remappings` 对各自的进程生效，不是修改整个 ROS 网络的全局名称。

### namespace 先了解一个边界

命名空间通常用于组织节点与相对通信名称。但是我们 C++ 中的 `/vehicle/status` 以 `/` 开头，是绝对名称。

因此仅给 `Node` 添加 `namespace='car1'`，不会自动把这个绝对话题变成 `/car1/vehicle/status`。不要把命名空间和重映射混为一谈。

本章实操先使用上面的明确重映射，不额外展开命名空间。

## 9.13 扩展：一起启动服务端和客户端

沿用第八章 `custom_interface_demo` 中的 `add_server` 和 `add_client`。停止车辆状态 Launch 后，新建：

```bash
nano ~/ros2_ws/src/custom_interface_demo/launch/three_ints.launch.py
```

完整文件：

```python
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    return LaunchDescription([
        Node(
            package='custom_interface_demo',
            executable='add_server',
            output='screen',
        ),
        Node(
            package='custom_interface_demo',
            executable='add_client',
            output='screen',
        ),
    ])
```

已有的目录安装规则会包含这个新文件，不需要再增加同样的 CMake 安装段。

```bash
cd ~/ros2_ws
colcon build --packages-up-to custom_interface_demo
source ~/ros2_ws/install/setup.bash
ros2 launch custom_interface_demo three_ints.launch.py
```

按第八章未修改的请求，预期客户端收到 `9` 后退出，服务端继续运行，所以 Launch 终端仍在等待。按 `Ctrl + C` 结束服务端。

### 为什么这次不需要手动等服务端

不是因为服务端写在列表前面，就保证已初始化完成。实际保障来自客户端中的：

```cpp
client->wait_for_service(std::chrono::seconds(1))
```

客户端先等服务可用，再发请求。若客户端先完成初始化，它可能短暂打印等待日志，然后正常调用。

不要为了服务启动顺序随意加固定延时；本例已有明确的就绪等待。

## 9.14 修改启动文件后要做什么

本章使用普通目录安装方式，没有启用符号链接安装。因此修改源码目录中的启动文件后：

1. 保存文件。
2. 重新执行构建，让安装规则更新安装目录中的副本。
3. 加载工作空间环境。
4. 停止旧启动，再运行新的 Launch。

这里的构建不意味着 Python 文件被编译成 C++ 可执行程序，而是让构建系统完成必要的安装步骤。

已经运行的程序不会因为你修改了启动文件，就自动更换名称或重启。

## 9.15 命令速查

| 用途 | 通式或本章例子 |
| --- | --- |
| 单独运行程序 | `ros2 run 包名 可执行程序名` |
| 运行包内启动文件 | `ros2 launch 包名 启动文件名` |
| 本章车辆状态启动 | `ros2 launch custom_interface_demo vehicle_status.launch.py` |
| 本章求和启动 | `ros2 launch custom_interface_demo three_ints.launch.py` |
| 构建及安装 | `colcon build --packages-up-to custom_interface_demo` |
| 加载工作空间 | `source ~/ros2_ws/install/setup.bash` |
| 检查节点 | `ros2 node list` |
| 检查原话题 | `ros2 topic info /vehicle/status` |
| 检查重映射话题 | `ros2 topic info /car/status` |
| 停止本次启动 | 在 Launch 终端按 `Ctrl + C` |

## 9.16 常见问题

| 现象 | 优先检查 |
| --- | --- |
| 找不到启动文件 | 文件名、目录安装规则、是否重新构建并加载环境 |
| 找不到可执行程序 | `executable` 是否填了构建目标名，而不是源码名或节点名 |
| Python 缩进或语法错误 | 四空格缩进、括号、逗号、引号是否完整，是否用了中文标点 |
| 找不到 `launch`、`launch_ros` 模块 | ROS 基础环境与依赖是否可用 |
| 发布者数量超过预期 | 是否仍有手动启动的旧发布者 |
| 节点重名 | 是否同时运行多份 Launch，或与其他程序用了相同名称 |
| 改启动文件仍是旧效果 | 是否保存、安装更新，并重新启动 |
| 重映射后不通信 | 双方是否都映射到同一名称，消息类型是否一致 |
| 客户端退出而终端仍在运行 | 服务端持续运行，属于本例设计 |

## 9.17 自检题与答案

先回答，再对照答案。

1. Launch 文件中 `executable='status_pub'`，能改成 `status_pub.cpp` 吗？
2. `name='vehicle_sender'` 改的是程序文件名还是运行时节点名？
3. 把两个程序写进 Launch 后，它们就变成一个节点了吗？
4. Launch 会替你执行求和公式或填写消息字段吗？
5. 为什么 `launch` 文件夹要添加到 CMake 安装规则？
6. 只重映射发布者，订阅者仍用旧话题，会正常接收到它的数据吗？
7. 服务端在列表第一项，就能保证客户端不用等待了吗？
8. 在 Launch 终端按 `Ctrl + C`，会关闭其他终端独立启动的所有 ROS 节点吗？

答案：

1. 不能这样替换，必须填写已安装可执行程序的名称。
2. 运行时节点名。
3. 没有。本例仍是两个独立进程，各自创建一个节点。
4. 不会，业务逻辑仍由 C++ 实现。
5. 让 `ros2 launch 包名 文件名` 能从包的安装目录中找到文件。
6. 不会，双方需要使用匹配的话题名称与类型。
7. 不能。客户端仍应等待服务就绪。
8. 不会，它只管理本次启动的相关进程。

## 9.18 完成本章的标准

- [ ] 能用一个 Launch 文件启动发布者、订阅者，并验证通信。
- [ ] 能解释 `package`、`executable`、`name` 的区别。
- [ ] 能说明启动文件与 C++ 业务代码各负责什么。
- [ ] 能修改安装配置，使包内启动文件可被找到。
- [ ] 能在 Launch 中改节点名，并检查结果。
- [ ] 能给通信双方设置相同的话题重映射。
- [ ] 能统一停止本次启动的节点。
- [ ] 能解释客户端等待服务与启动顺序之间的关系。

本章先掌握基本启动、改名、重映射。参数文件、可配置启动参数、条件启动和事件处理可以后续按项目需要逐步扩展。

## 参考资料与验证范围

- [ROS 2 Jazzy：创建 Launch 文件](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Launch/Creating-Launch-Files.html)
- [ROS 2 Jazzy：将 Launch 文件集成到包](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Launch/Launch-system.html)
- [创建教程的 ROS 文档预览站](https://repo.test.ros2.org/en/jazzy/Tutorials/Intermediate/Launch/Creating-Launch-Files.html)
- [包集成教程的 ROS 文档预览站](https://repo.test.ros2.org/en/jazzy/Tutorials/Intermediate/Launch/Launch-system.html)

启动文件形式与安装方式已对照 Jazzy 官方教程核对。本文案例依据你第八章已运行成功的程序设计；本次未在你的 Ubuntu 虚拟机中实际执行，启动与通信验证将在逐步实操时完成。
