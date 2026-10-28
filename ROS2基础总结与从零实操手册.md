# ROS 2 基础总结与从零实操手册

> 面向第一次独立搭建项目的学习者。
> 环境：Windows 上的 VirtualBox → Ubuntu 24.04 → ROS 2 Jazzy → C++17。
> 建议把本手册加入 ROS2study。正文是一套复习练习，最后的任务书才是由你独立完成的小项目。
> 不要求一次读完，更不要求默写全部代码。每节先理解、操作、验证，再继续。

## 0. 这份手册怎样用

本手册把前十章的项目基础串成一个可从零建立的练习：

- 自己定义车辆状态消息和求和服务类型。
- 写一个按参数发布车辆状态的程序。
- 写一个收到车辆状态就打印的程序。
- 写求和服务端和客户端。
- 配置依赖、构建并运行。
- 用 YAML 配置参数，用 Launch 启动程序。
- 用命令验证数据和排查故障。

这套练习放在全新的 `~/ros2_review_ws` 工作空间，不覆盖以前的 `~/ros2_ws`。接口和程序也使用新名字，减少和旧练习混淆。

本手册所说的“全部基础”，指完成本手册练习及末尾第一个模拟车辆小项目需要的知识。它不是 ROS 2 所有功能的百科全书。TF2、真实机器人、导航和 C++ 动作端点实现不属于本次项目的前置要求。

### 推荐节奏

| 次序 | 本次完成什么 | 结束时应看到什么 |
| --- | --- | --- |
| 第一段 | 认识目录、创建工作空间和接口包 | 能查看自己定义的 msg、srv |
| 第二段 | 编写四个 C++ 文件及构建配置 | 程序包构建成功 |
| 第三段 | 单独启动程序并查改数据 | 话题通信、服务返回、参数更新都正常 |
| 第四段 | YAML、Launch、重映射、排错 | 一条命令启动，能解释配置如何生效 |
| 第五段 | 脱离逐句指令做验收 | 能自己定位文件、修改并验证 |

每次结束，在运行中的程序窗口按 `Ctrl + C` 停止。下次不用重新建包，加载环境后从未完成的步骤继续。

## 1. 最重要的概念先放在一起

### 1.1 工作空间、包、程序、节点

| 概念 | 本手册例子 | 含义 |
| --- | --- | --- |
| 工作空间 | `~/ros2_review_ws` | 一组功能包及其构建产物所在目录 |
| 功能包 | `review_interfaces`、`review_nodes` | 组织源码、接口、配置和依赖 |
| 源文件 | `status_publisher.cpp` | 给编译器看的 C++ 源代码 |
| 可执行程序 | `status_publisher` | 构建后可以运行的程序 |
| 进程 | 程序运行后的操作系统进程 | 系统分配内存、调度的对象 |
| 节点 | `/review_publisher` | 运行时参与 ROS 通信的对象 |

一个包可以有多个程序，一个进程可以创建多个节点。本手册每个程序创建一个节点，方便学习；不要把这种简化当成 ROS 的硬性限制。

```bash
ros2 run review_nodes status_publisher
```

表示运行 `review_nodes` 包中的 `status_publisher` 程序。节点名称由程序创建节点时指定，也可以在启动时覆盖。

### 1.2 通信与配置的区别

| 机制 | 角色和数据 | 本手册或前章例子 |
| --- | --- | --- |
| 话题 Topic | 发布者发消息，订阅者处理消息，无须逐条业务应答 | 持续发布车辆状态 |
| 服务 Service | 客户端发请求，服务端处理并返回响应 | 请求计算两数之和 |
| 动作 Action | 客户端提出目标，服务端管理任务，可有反馈、结果与取消请求 | 海龟转到目标朝向 |
| 参数 Parameter | 节点持有的配置值，程序按自己的逻辑使用 | 发布的模拟速度 |

话题可以只发布一条消息，服务也可以多次调用。选择机制要看交互需要，不能只按“发几次”或“快慢”机械判断。

### 1.3 名称、类型、字段、对象、值

| 概念 | 本手册例子 |
| --- | --- |
| 话题名称 | `/review/vehicle_status` |
| 消息类型 | `review_interfaces/msg/VehicleStatus` |
| 消息字段 | `speed` |
| C++ 对象 | `VehicleStatus message;` 中的 `message` |
| 具体值 | `message.speed = 5.0;` 中的 `5.0` |

`.msg` 定义字段名称和类型，不保存某辆车实时测到的速度。程序创建对象、填写字段，再发布出去。

## 2. 先分清在哪个系统操作

### 2.1 Windows 与 Ubuntu 不共用同一套路径

Windows 的：

```text
F:\ROS2study
```

是保存 Markdown 笔记的仓库。

Ubuntu 的：

```text
/home/tianhong/ros2_review_ws
```

是接下来创建练习代码的工作空间。两者是不同系统中的目录。上传了笔记，不表示已经备份 Ubuntu 的代码。

除专门说明外，本手册所有命令都在 Ubuntu 终端执行，不在 Windows PowerShell 执行。

### 2.2 打开终端并认识提示符

启动 Ubuntu，登录后按 `Ctrl + Alt + T`。

提示符可能是：

```text
tianhong@ros-vm:~$
```

只输入 `$` 后面的命令，不复制提示符。`~` 表示当前用户主目录，本机通常是 `/home/tianhong`。

代码块中三条命令就是三条命令，建议逐条按 Enter，观察结果后再继续。

## 3. Ubuntu 操作小抄

| 命令 | 作用 | 例子 |
| --- | --- | --- |
| `pwd` | 当前位于哪里 | `pwd` |
| `ls` | 当前目录有什么 | `ls` |
| `ls 路径` | 指定目录有什么 | `ls ~/ros2_review_ws/src` |
| `cd 路径` | 切换目录 | `cd ~/ros2_review_ws` |
| `cd ..` | 返回父目录 | 当前在包目录时回到上一层 |
| `mkdir -p 路径` | 创建文件夹及缺失的父目录，已存在则保留 | `mkdir -p ~/ros2_review_ws/src` |
| `nano 文件路径` | 编辑已有文件，或准备新建文件 | `nano ~/notes.txt` |
| `cat 文件路径` | 在终端显示文本文件 | `cat ~/notes.txt` |
| `cp 源文件 目标文件` | 复制，常用于修改前备份 | `cp CMakeLists.txt CMakeLists.backup.txt` |

`mkdir` 创建的是文件夹；`nano` 编辑并保存的是文件。用 nano 保存新文件前，其父文件夹必须存在。

`~/` 从主目录开始找，`./` 从当前目录开始找。以 `/` 开头的是绝对路径。Linux 区分大小写，`VehicleStatus.msg` 和 `vehiclestatus.msg` 不同。

路径有空格时加英文引号。本练习名称不含空格，便于输入。

### 3.1 nano 的详细操作

1. 输入 `nano 文件路径` 后按 Enter。
2. 编辑器打开后，粘贴或输入代码。
3. 按 `Ctrl + O`，是英文字母 O，不是数字 0。
4. 底部询问保存文件名，保持正确路径，按 Enter。
5. 按 `Ctrl + X` 退出，回到终端提示符。

Ubuntu 终端粘贴通常用 `Ctrl + Shift + V`。Windows 到虚拟机的复制还需要 VirtualBox 共享剪贴板可用。

nano 中 `Ctrl + K` 剪切当前行，`Ctrl + U` 粘回剪切内容；需要整体替换短配置时，可以反复 `Ctrl + K` 删除旧行后再粘贴。`Alt + U` 通常撤销一次编辑，`Alt + E` 重做；以底部帮助和当前 nano 版本为准。

不要把终端里的 `Ctrl + C` 当成复制。在运行程序的终端中，它通常是停止前台程序；nano 保存仍按上面的 `Ctrl + O`。

新文件只有保存后才真正写入磁盘。看到空编辑器不代表代码已经存在。

## 4. 环境和构建：先理解再执行

### 4.1 加载基础 ROS 环境

```bash
source /opt/ros/jazzy/setup.bash
echo $ROS_DISTRO
```

预期输出 `jazzy`。`source` 把脚本中的环境设置加载到当前终端，不是编译或启动节点。

新开的终端可能由 `.bashrc` 自动加载基础环境。手动执行上面的 source 可以明确当前步骤；仅凭输出 jazzy 不能判断究竟是谁加载了它。

### 4.2 三个动作分清

| 动作 | 作用 |
| --- | --- |
| `colcon build` | 组织各包的构建与安装 |
| `source 工作空间/install/setup.bash` | 让当前终端找到已安装的包和资源 |
| `ros2 run` / `ros2 launch` | 启动程序 |

CMake 根据 `CMakeLists.txt` 配置规则，底层构建工具和 C++ 编译器生成程序；colcon 组织包的构建。它们不是同一个编译器的不同名字。

构建终端优先加载基础 ROS 环境，不主动加载正在构建的工作空间；运行终端再加载工作空间。后面把两种终端分开操作。

## 5. 从零创建新工作空间

在构建终端执行：

```bash
source /opt/ros/jazzy/setup.bash
mkdir -p ~/ros2_review_ws/src
cd ~/ros2_review_ws
pwd
ls
```

`pwd` 应为 `/home/tianhong/ros2_review_ws`，`ls` 应有 `src`。如果已经有旧内容，先检查，不删除它。

工作空间的 `src` 放包，包内的 `src` 放 `.cpp`，是两层不同目录。

## 6. 创建接口包

```bash
cd ~/ros2_review_ws/src
ros2 pkg create review_interfaces --build-type ament_cmake --license Apache-2.0
mkdir -p ~/ros2_review_ws/src/review_interfaces/msg
mkdir -p ~/ros2_review_ws/src/review_interfaces/srv
```

创建包只生成基础目录和配置，不会自动实现节点、安装全部依赖或完成构建。

本包只定义接口，不编写节点程序，所以此处不添加 rclcpp。

### 6.1 创建消息文件

```bash
nano ~/ros2_review_ws/src/review_interfaces/msg/VehicleStatus.msg
```

完整内容：

```text
string vehicle_name
float64 speed
bool stopped
```

保存退出。每行是“类型 字段名”，末尾不加 C++ 分号。速度单位约定为 m/s，接口不会自动校验单位或自动测量停车状态。

### 6.2 创建服务文件

```bash
nano ~/ros2_review_ws/src/review_interfaces/srv/AddTwoInts.srv
```

完整内容：

```text
int64 a
int64 b
---
int64 sum
```

保存退出。上面是请求字段，下面是响应字段。它只定义结构，求和由服务端 C++ 回调实现。

### 6.3 接口包 CMakeLists.txt

```bash
nano ~/ros2_review_ws/src/review_interfaces/CMakeLists.txt
```

替换为完整内容：

```cmake
cmake_minimum_required(VERSION 3.8)
project(review_interfaces)

find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/VehicleStatus.msg"
  "srv/AddTwoInts.srv"
)

ament_export_dependencies(rosidl_default_runtime)
ament_package()
```

`rosidl_generate_interfaces` 安排生成 C++、Python 等使用的接口代码。`${PROJECT_NAME}` 是上面的 `review_interfaces`。没有 `add_executable`，因为这个包没有节点可执行程序。

### 6.4 接口包 package.xml

```bash
nano ~/ros2_review_ws/src/review_interfaces/package.xml
```

为避免第一次手工插入标签出错，本练习使用下面完整文件替换：

```xml
<?xml version="1.0"?>
<package format="3">
  <name>review_interfaces</name>
  <version>0.0.0</version>
  <description>Interfaces for ROS 2 review exercises</description>
  <maintainer email="tianhong@example.com">tianhong</maintainer>
  <license>Apache-2.0</license>

  <buildtool_depend>ament_cmake</buildtool_depend>
  <buildtool_depend>rosidl_default_generators</buildtool_depend>
  <exec_depend>rosidl_default_runtime</exec_depend>
  <member_of_group>rosidl_interface_packages</member_of_group>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

邮箱是练习用元数据，可换成自己的维护信息。不要在已有 `<package>` 里再嵌套整份文件。

`package.xml` 说明包的信息和依赖；CMake 说明具体生成、构建和安装哪些内容。两者需要对应，不是只写一个就可以替代另一个。

### 6.5 构建接口包

构建终端：

```bash
cd ~/ros2_review_ws
colcon build --packages-select review_interfaces
```

确认 Summary 成功。若有错误先停下来检查，不继续运行后续步骤。

新开“运行/检查终端”，执行：

```bash
source /opt/ros/jazzy/setup.bash
source ~/ros2_review_ws/install/setup.bash
ros2 interface show review_interfaces/msg/VehicleStatus
ros2 interface show review_interfaces/srv/AddTwoInts
```

应显示自己刚写的字段。现在只有类型，还没有发布者、订阅者或服务端在运行。

### 6.6 类型的三种写法

| 场合 | 消息类型示例 |
| --- | --- |
| ROS 命令 | `review_interfaces/msg/VehicleStatus` |
| C++ 类型 | `review_interfaces::msg::VehicleStatus` |
| C++ 头文件 | `review_interfaces/msg/vehicle_status.hpp` |

头文件由构建生成，不自己再写一个同名 `.hpp`，也不手动改安装目录中的生成文件。

## 7. 创建程序包

回到构建终端：

```bash
cd ~/ros2_review_ws/src
ros2 pkg create review_nodes --build-type ament_cmake --license Apache-2.0 --dependencies rclcpp review_interfaces
mkdir -p ~/ros2_review_ws/src/review_nodes/launch
mkdir -p ~/ros2_review_ws/src/review_nodes/config
```

`review_interfaces` 是本地接口依赖，后面 `--packages-up-to` 会安排它先构建。建包命令声明依赖，不等于会自动从网上下载这个自定义包。

本手册将创建以下四个源文件。先逐个保存，暂时不要构建，等配置齐了再一起构建。

| 源文件 | 可执行程序 | 运行时默认节点名 |
| --- | --- | --- |
| `status_publisher.cpp` | `status_publisher` | `/review_publisher` |
| `status_subscriber.cpp` | `status_subscriber` | `/review_subscriber` |
| `add_server.cpp` | `add_server` | `/review_add_server` |
| `add_client.cpp` | `add_client` | `/review_add_client` |

## 8. 发布者：参数怎样成为消息

### 8.1 创建并保存文件

```bash
nano ~/ros2_review_ws/src/review_nodes/src/status_publisher.cpp
```

完整代码：

```cpp
#include <chrono>
#include <memory>
#include <string>

#include "rclcpp/rclcpp.hpp"
#include "review_interfaces/msg/vehicle_status.hpp"

using VehicleStatus = review_interfaces::msg::VehicleStatus;

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  auto node = std::make_shared<rclcpp::Node>("review_publisher");

  node->declare_parameter<std::string>("vehicle_name", "review_car");
  node->declare_parameter<double>("speed", 5.0);

  auto publisher = node->create_publisher<VehicleStatus>(
    "/review/vehicle_status", 10);

  auto publish_status = [node, publisher]() {
    std::string vehicle_name =
      node->get_parameter("vehicle_name").as_string();
    double speed = node->get_parameter("speed").as_double();

    VehicleStatus message;
    message.vehicle_name = vehicle_name;
    message.speed = speed;
    message.stopped = (speed == 0.0);

    publisher->publish(message);

    RCLCPP_INFO(
      node->get_logger(), "Published: %s, %.1f m/s, stopped=%s",
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

保存退出后再看解释。

### 8.2 这些代码分别负责什么

| 代码 | 作用 |
| --- | --- |
| `rclcpp::init(argc, argv)` | 初始化 ROS 上下文并处理 ROS 启动参数 |
| `std::make_shared<rclcpp::Node>(...)` | 创建节点，返回智能指针 |
| `declare_parameter` | 声明节点提供的配置及默认值 |
| `create_publisher<VehicleStatus>` | 创建该消息类型的发布接口 |
| `get_parameter` | 读取当前配置 |
| `VehicleStatus message` | 创建一个消息对象 |
| `message.speed = speed` | 填写这次发送的速度 |
| `publisher->publish(message)` | 真正发布业务消息 |
| `RCLCPP_INFO` | 打印信息日志 |
| `create_wall_timer` | 周期性安排回调 |
| `spin(node)` | 让执行器等待并处理就绪工作 |
| `shutdown()` | 关闭当前 ROS 上下文 |

### 8.3 新手最容易混淆的语法

- `using VehicleStatus = ...` 是类型别名，不是创建对象。
- `auto` 让编译器推导类型，不是没有类型。
- `node`、`publisher` 是智能指针，使用 `->` 调用对象成员。
- `message` 是对象，使用 `.` 访问字段。
- `[node, publisher]` 把外部两个智能指针按值捕获到 lambda 中，方便回调使用。
- `()` 是回调的参数列表；本例不接收参数。
- `{...}` 是将来被调用时执行的函数体。

定义 lambda 不会立刻发布消息。定时器到期、执行器处理它时才执行。

`10` 是历史深度设置，不是 10 Hz。本例发送周期由 `seconds(1)` 决定。

保留 `timer`、`publisher` 等对象，使其在需要的期间保持存在。

### 8.4 参数如何起作用

本例每次回调都重新读取参数，再把值填进消息。因此运行时更新 `speed`，后续回调能使用新值。

如果写死 `message.speed = 5.0;`，或者仅启动时读取并一直使用旧缓存，外部参数更新就不会自动改变这里的输出。

`stopped` 是代码根据本例模拟速度是否精确为零计算出来的，不是 ROS 自动测量停车。该程序仅供模拟练习，没有速度范围和非有限数校验；本手册用有限、非负数值操作。

## 9. 订阅者：收到消息后执行什么

```bash
nano ~/ros2_review_ws/src/review_nodes/src/status_subscriber.cpp
```

完整代码：

```cpp
#include <memory>

#include "rclcpp/rclcpp.hpp"
#include "review_interfaces/msg/vehicle_status.hpp"

using VehicleStatus = review_interfaces::msg::VehicleStatus;

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  auto node = std::make_shared<rclcpp::Node>("review_subscriber");

  auto receive_status = [node](VehicleStatus::ConstSharedPtr message) {
    RCLCPP_INFO(
      node->get_logger(), "Received: %s, %.1f m/s, stopped=%s",
      message->vehicle_name.c_str(), message->speed,
      message->stopped ? "true" : "false");
  };

  auto subscription = node->create_subscription<VehicleStatus>(
    "/review/vehicle_status", 10, receive_status);

  rclcpp::spin(node);
  rclcpp::shutdown();
  return 0;
}
```

收到兼容消息并由执行器处理时，会调用 `receive_status`。它没有定时器，不会独立每秒打印一次。

`ConstSharedPtr` 指向只读消息对象，所以用 `message->speed`。不同进程中同名的 message 变量不是同一个本地对象，也不是把发布端的局部指针传给了订阅端。

发布者即使没有订阅者也能 publish。默认 volatile 持久性不会保证后来启动的订阅者补收到全部历史消息。双方除了名称、类型一致，还需 QoS 兼容、发现和通信环境正常。

## 10. 服务端：处理请求并填写响应

```bash
nano ~/ros2_review_ws/src/review_nodes/src/add_server.cpp
```

完整代码：

```cpp
#include <memory>

#include "rclcpp/rclcpp.hpp"
#include "review_interfaces/srv/add_two_ints.hpp"

using AddTwoInts = review_interfaces::srv::AddTwoInts;

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  auto node = std::make_shared<rclcpp::Node>("review_add_server");

  auto handle_request = [node](
    std::shared_ptr<AddTwoInts::Request> request,
    std::shared_ptr<AddTwoInts::Response> response)
  {
    response->sum = request->a + request->b;
    RCLCPP_INFO(
      node->get_logger(), "Calculated sum: %lld",
      static_cast<long long>(response->sum));
  };

  auto service = node->create_service<AddTwoInts>(
    "/review/add_two_ints", handle_request);

  RCLCPP_INFO(node->get_logger(), "Service ready: /review/add_two_ints");
  rclcpp::spin(node);
  rclcpp::shutdown();
  return 0;
}
```

`create_service` 创建接口并关联回调，不会立刻执行求和。请求到达并被处理时，ROS 调用回调并提供请求与响应对象。填写响应、回调完成后，由 ROS 将响应传回客户端。

日志打印不是响应本身。删去日志但保留求和，仍然可以返回结果。

`%lld` 配合转换成 `long long` 的值打印整数。本例只用小整数练习，没有处理整型求和溢出。

## 11. 客户端：请求、等待、取结果

```bash
nano ~/ros2_review_ws/src/review_nodes/src/add_client.cpp
```

完整代码：

```cpp
#include <chrono>
#include <memory>

#include "rclcpp/rclcpp.hpp"
#include "review_interfaces/srv/add_two_ints.hpp"

using AddTwoInts = review_interfaces::srv::AddTwoInts;

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  auto node = std::make_shared<rclcpp::Node>("review_add_client");
  auto client = node->create_client<AddTwoInts>("/review/add_two_ints");

  while (!client->wait_for_service(std::chrono::seconds(1))) {
    if (!rclcpp::ok()) {
      rclcpp::shutdown();
      return 0;
    }
    RCLCPP_INFO(node->get_logger(), "Waiting for service...");
  }

  auto request = std::make_shared<AddTwoInts::Request>();
  request->a = 2;
  request->b = 3;

  auto future = client->async_send_request(request);
  auto status = rclcpp::spin_until_future_complete(
    node, future, std::chrono::seconds(10));

  if (status == rclcpp::FutureReturnCode::SUCCESS) {
    auto response = future.get();
    RCLCPP_INFO(
      node->get_logger(), "Received sum: %lld",
      static_cast<long long>(response->sum));
  } else {
    RCLCPP_ERROR(node->get_logger(), "Response timed out or was interrupted.");
    client->remove_pending_request(future);
    rclcpp::shutdown();
    return 1;
  }

  rclcpp::shutdown();
  return 0;
}
```

| 阶段 | 代码 | 含义 |
| --- | --- | --- |
| 创建接口 | `create_client` | 准备调用指定服务，还没发请求 |
| 等服务 | `wait_for_service` | 服务端是否可用 |
| 填请求 | `request->a = 2` | 准备本次请求的数据 |
| 发请求 | `async_send_request` | 真正提出请求 |
| 等响应 | `spin_until_future_complete` | 处理工作并等待这次请求的响应 |
| 判断状态 | `status == SUCCESS` | 是否成功完成等待 |
| 取响应 | `future.get()` | 得到响应对象的智能指针 |
| 读答案 | `response->sum` | 取出计算结果 |

一秒是本例单次等服务的时长，循环可以继续等待；十秒是请求发送后等待响应的超时。它们不是同一件事。

超时清理 `remove_pending_request` 不表示取消服务端已经开始的操作。服务没有动作那样的标准任务取消流程。

本例客户端收到一次响应就退出；服务端继续运行。这由我们写的代码决定，并不是所有客户端必须立即退出。

## 12. 配置程序包的构建

```bash
nano ~/ros2_review_ws/src/review_nodes/CMakeLists.txt
```

完整内容：

```cmake
cmake_minimum_required(VERSION 3.8)
project(review_nodes)

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(review_interfaces REQUIRED)

add_executable(status_publisher src/status_publisher.cpp)
target_compile_features(status_publisher PUBLIC cxx_std_17)
ament_target_dependencies(status_publisher rclcpp review_interfaces)

add_executable(status_subscriber src/status_subscriber.cpp)
target_compile_features(status_subscriber PUBLIC cxx_std_17)
ament_target_dependencies(status_subscriber rclcpp review_interfaces)

add_executable(add_server src/add_server.cpp)
target_compile_features(add_server PUBLIC cxx_std_17)
ament_target_dependencies(add_server rclcpp review_interfaces)

add_executable(add_client src/add_client.cpp)
target_compile_features(add_client PUBLIC cxx_std_17)
ament_target_dependencies(add_client rclcpp review_interfaces)

install(TARGETS
  status_publisher
  status_subscriber
  add_server
  add_client
  DESTINATION lib/${PROJECT_NAME}
)

install(DIRECTORY launch config
  DESTINATION share/${PROJECT_NAME}
)

ament_package()
```

`launch`、`config` 文件夹已经创建，即使目前为空也可以安装，文件稍后写入。

| 配置 | 作用 |
| --- | --- |
| `find_package` | 查找依赖提供的构建信息 |
| `add_executable` | 定义程序目标及使用哪个源文件 |
| `target_compile_features` | 声明 C++17 要求 |
| `ament_target_dependencies` | 给程序配置 ROS 包依赖 |
| `install(TARGETS ...)` | 把程序安装到 ROS 能查找的位置 |
| `install(DIRECTORY ...)` | 把资源文件夹安装到共享目录 |
| `ament_package()` | 完成包配置与导出，通常放末尾 |

### 12.1 程序包的 package.xml

```bash
nano ~/ros2_review_ws/src/review_nodes/package.xml
```

替换为完整内容：

```xml
<?xml version="1.0"?>
<package format="3">
  <name>review_nodes</name>
  <version>0.0.0</version>
  <description>Nodes for ROS 2 review exercises</description>
  <maintainer email="tianhong@example.com">tianhong</maintainer>
  <license>Apache-2.0</license>

  <buildtool_depend>ament_cmake</buildtool_depend>
  <depend>rclcpp</depend>
  <depend>review_interfaces</depend>
  <exec_depend>launch</exec_depend>
  <exec_depend>launch_ros</exec_depend>
  <exec_depend>ros2launch</exec_depend>
  <exec_depend>ament_index_python</exec_depend>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

`depend` 说明程序需要使用对应包；`exec_depend` 声明运行启动文件时要用的工具。这些声明不等于软件已经安装，但你前十章环境通常已经具备这些依赖。

## 13. 构建程序包

回到构建终端：

```bash
cd ~/ros2_review_ws
colcon build --packages-up-to review_nodes
```

`--packages-up-to` 选择目标包及其工作空间内依赖，先接口包、再程序包。`--packages-select` 只选择指定包，依赖需要另外已经可用。

本练习预期 Summary 显示两个包成功。一个包有四个可执行程序，仍然只算一个包。

构建后工作空间有：

| 目录 | 放什么 | 手动编辑吗 |
| --- | --- | --- |
| `src` | 自己维护的包、源码和配置 | 是 |
| `build` | 中间构建文件 | 通常不编辑 |
| `install` | 安装的程序、接口和资源 | 不把它当源码编辑 |
| `log` | 构建日志 | 用于查错 |

检查终端重新加载新工作空间：

```bash
source /opt/ros/jazzy/setup.bash
source ~/ros2_review_ws/install/setup.bash
ros2 pkg executables review_nodes
```

应能查到本章四个可执行程序。若查询不到，先检查构建结果与安装规则，不直接重装 ROS。

## 14. 第一次验证：发布和订阅

每个新的运行终端都执行下面两句加载环境，后面不再每次重复写出：

```bash
source /opt/ros/jazzy/setup.bash
source ~/ros2_review_ws/install/setup.bash
```

终端 A 运行：

```bash
ros2 run review_nodes status_publisher
```

应每秒左右打印 `review_car`、`5.0 m/s`、`stopped=false`。终端不返回提示符是程序正在运行，不是编辑器卡住。

终端 B 加载环境后运行：

```bash
ros2 run review_nodes status_subscriber
```

应看到相同内容的 `Received` 日志。

终端 C 加载环境后检查：

```bash
ros2 node list
ros2 node info /review_publisher
ros2 topic list -t
ros2 topic info /review/vehicle_status
ros2 interface show review_interfaces/msg/VehicleStatus
ros2 topic echo --once /review/vehicle_status
```

预期消息：

```yaml
vehicle_name: review_car
speed: 5.0
stopped: false
```

此时仅有这两个业务程序、没有额外查询订阅运行时，话题应有一个发布端和一个订阅端。`echo`、`hz` 本身也是临时订阅者，观察过程中数量可能增加。

### 14.1 做一个预测实验

先自己预测，再操作：

1. 订阅者按 `Ctrl + C`，发布者会停止吗？不会。
2. 重启订阅者，能收到新消息吗？能，只要通信条件正常。
3. 发布者按 `Ctrl + C`，订阅者会退出吗？本例不会，它继续等待。

这不是故意让你重复操作，而是确认你知道程序为何运行、何时执行回调。

### 14.2 检查频率

```bash
ros2 topic hz /review/vehicle_status
```

它统计该工具实际收到消息的频率，可以辅助检查周期，但不直接等于发布者的定时器频率。虚拟机调度、QoS、接收开销等都可能影响结果。观察结束按 `Ctrl + C`。

## 15. 第二次验证：运行时改参数

保持发布者运行，检查终端执行：

```bash
ros2 param list /review_publisher
ros2 param get /review_publisher speed
ros2 param set /review_publisher speed 12.0
ros2 param get /review_publisher speed
ros2 topic echo --once /review/vehicle_status
```

设置成功后，随后发布的消息 speed 应为 `12.0`。刚更新的瞬间可能还观察到前一周期的值，等下一次回调再检查。

再执行：

```bash
ros2 param set /review_publisher speed 0.0
```

随后应看到 `speed=0.0`、`stopped=true`。没有重新编译，这是参数被程序重新读取使用的效果。

如果只改参数却保留写死的赋值，这个效果不会自动出现。

### 15.1 默认值、启动值、运行值

- 未指定启动覆盖：使用 C++ 默认值 `5.0`。
- 启动时加载 YAML 或 `-p`：使用匹配的启动设置。
- 运行时 `param set`：更新当前节点配置，本例下次回调使用新值。
- 关闭后默认启动：此前运行时改动不会自动保留。

`param set` 不会自动修改源码或 YAML。

## 16. 第三次验证：服务端和客户端

先停止不需要的发布、订阅练习，避免窗口过多。

终端 A 加载环境后：

```bash
ros2 run review_nodes add_server
```

终端 B 加载环境后先用命令测试：

```bash
ros2 service list -t
ros2 service type /review/add_two_ints
ros2 interface show review_interfaces/srv/AddTwoInts
ros2 service call /review/add_two_ints review_interfaces/srv/AddTwoInts "{a: 10, b: 7}"
```

应返回 `sum=17`。这时命令行就是客户端。

再运行自己的 C++ 客户端：

```bash
ros2 run review_nodes add_client
```

应返回 `Received sum: 5`，因为客户端源码填写的是 `2`、`3`。命令行上次发送 `10`、`7` 不会修改客户端源码。

客户端正常结束后回到提示符，服务端仍在运行。事后 `node list` 看不到短暂运行的客户端，不代表它没有启动过。

### 16.1 服务就绪实验

停止服务端，先运行客户端，应反复打印等待日志；再在另一终端运行服务端，客户端应继续发送并取得结果。

这是 wait_for_service 的作用。不要把“服务端写在 Launch 列表前面”当作就绪保证。

## 17. 改代码后怎样更新

练习把客户端请求改成 `a=6`、`b=9`：

1. 打开 `~/ros2_review_ws/src/review_nodes/src/add_client.cpp`。
2. 修改对应赋值，不要改 `.srv`，因为字段结构没变。
3. 保存退出。
4. 构建终端执行：

```bash
cd ~/ros2_review_ws
colcon build --packages-up-to review_nodes
```

5. 运行终端重新加载工作空间并运行客户端：

```bash
source ~/ros2_review_ws/install/setup.bash
ros2 run review_nodes add_client
```

预期 `15`。若服务端源码没有变，可以继续保留原服务端进程。

新增字段则不同：需要改接口、生成类型、调整相关程序并重新构建、重启使用新类型的程序。类型定义变更后，不让通信双方继续各用不同结构。

## 18. 创建 YAML 配置

停止之前单独启动的练习程序。

```bash
nano ~/ros2_review_ws/src/review_nodes/config/vehicle.yaml
```

完整内容：

```yaml
/review_publisher:
  ros__parameters:
    vehicle_name: "yaml_car"
    speed: 8.0
```

第一行是目标节点名，不是包名、程序名或车辆名称。`ros__parameters` 中间有两个下划线，每层用两个空格，不使用 Tab。

`speed` 是浮点参数，写 `8.0`，而不是整数 `8`。

先直接验证这个文件：

```bash
ros2 run review_nodes status_publisher --ros-args --params-file ~/ros2_review_ws/src/review_nodes/config/vehicle.yaml
```

预期输出 `yaml_car`、`8.0`。这里直接读取源码路径中的配置文件，所以只保存 YAML 就可以运行，不需要先安装它。

确认后按 `Ctrl + C` 停止。

## 19. 创建 Launch：统一启动并加载配置

```bash
nano ~/ros2_review_ws/src/review_nodes/launch/review.launch.py
```

完整内容：

```python
import os

from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    package_share = get_package_share_directory('review_nodes')
    parameter_file = os.path.join(package_share, 'config', 'vehicle.yaml')

    return LaunchDescription([
        Node(
            package='review_nodes',
            executable='status_publisher',
            name='review_publisher',
            parameters=[parameter_file],
            output='screen',
        ),
        Node(
            package='review_nodes',
            executable='status_subscriber',
            name='review_subscriber',
            output='screen',
        ),
        Node(
            package='review_nodes',
            executable='add_server',
            output='screen',
        ),
    ])
```

### 19.1 逐项解释

| 内容 | 含义 |
| --- | --- |
| `import` / `from ... import ...` | 引入 Python 工具 |
| `get_package_share_directory` | 找到包安装后的共享目录 |
| `os.path.join` | 拼出配置文件路径 |
| `generate_launch_description` | ROS 加载 Python Launch 时调用的入口函数 |
| `LaunchDescription` | 保存本次启动动作 |
| `Node` | 描述启动一个 ROS 程序的动作 |
| `package` | 到哪个包里找程序 |
| `executable` | 启动哪个已安装的可执行程序 |
| `name` | 覆盖本次运行的节点名称 |
| `parameters` | 交给节点的参数配置 |
| `output='screen'` | 输出到当前终端 |

Python 用缩进组织代码，函数体用四个空格，不加 C++ 大括号或分号。

Launch 的 Node 动作不是 C++ 中实际创建节点的那一句。它先启动可执行程序，程序再按自己的代码创建节点。

这次启动发布者、订阅者、服务端三个进程，每个程序创建一个节点。Launch 不会把它们合并，也不代替它们的计算和通信。

故意没有把客户端一直放进 Launch：客户端可在另一个终端按需调用，更容易观察每次请求。后续也可以把 add_client 加为第四个启动动作，它完成一次请求后就退出，其他程序继续运行。

### 19.2 更新安装并启动

构建终端：

```bash
cd ~/ros2_review_ws
colcon build --packages-up-to review_nodes
```

因为 CMake 已有 `install(DIRECTORY launch config ...)`，这次会安装新文件，无须重复添加规则。

运行终端：

```bash
source /opt/ros/jazzy/setup.bash
source ~/ros2_review_ws/install/setup.bash
ros2 launch review_nodes review.launch.py
```

应看到三个进程启动，发布者和订阅者都显示 `yaml_car`、`8.0`。

检查终端加载环境后：

```bash
ros2 node list
ros2 topic info /review/vehicle_status
ros2 param get /review_publisher speed
ros2 service call /review/add_two_ints review_interfaces/srv/AddTwoInts "{a: 4, b: 5}"
```

预期三个节点、一个状态发布端、一个状态订阅端，参数 `8.0`，求和响应 `9`。

### 19.3 停止

在 Launch 终端按 `Ctrl + C`，它会向本次管理的进程发出停止请求；本例程序正常响应后退出。

不会因此关闭其他终端独立启动的所有 ROS 程序。短暂的发现信息更新延迟也可能使查询还显示刚退出的节点，稍等再检查。

## 20. 改节点名、重映射、改参数有什么不同

### 20.1 改节点名

Launch 的 `name='review_subscriber'` 改成 `name='my_receiver'` 后，节点列表显示 `/my_receiver`，可执行程序仍叫 `status_subscriber`，话题不因此自动改名。

如果改的是发布者节点名，YAML 中匹配发布者的节点键也要同步检查，否则文件可能不匹配而使用默认值。

### 20.2 话题重映射

把本例话题换到 `/review/car_status` 时，在发布者和订阅者两个 Node 动作中都加入：

```python
remappings=[('/review/vehicle_status', '/review/car_status')],
```

顺序是“原名、新名”。程序这次运行直接使用新名称，不是先发到旧话题再转发。消息结构不变。

只改发布者时，订阅者还监听旧话题，两者不能按原对应关系通信。重映射对配置它的进程生效，不是全网络一起改名。

保存后重新构建更新 Launch 安装副本，再停止旧启动、运行新启动，检查：

```bash
ros2 topic info /review/car_status
ros2 topic echo --once /review/car_status
```

做完这个选做实验后，删除双方刚加的 remappings 行，重新安装、重启，恢复后文的原话题名。不要把“查旧名无数据”误认为程序坏了。

### 20.3 绝对名称与命名空间

本例话题以 `/` 开头，是绝对名称。仅给 Node 增加 namespace，不会自动给这个绝对话题加前缀。命名空间主要组织节点及相对名称；先掌握明确的重映射即可。

### 20.4 对照表

| 改动 | 改什么 | 程序逻辑是否自动新增 |
| --- | --- | --- |
| `name='...'` | 节点名称 | 不会 |
| `remappings` | 运行时通信名称 | 不会 |
| 参数 speed | 当前配置值，需代码使用 | 不会 |
| `.msg` 新增字段 | 数据结构 | 不会自动写获取或处理逻辑 |

## 21. 配置的保存与更新

### 21.1 导出运行中的参数

节点运行时：

```bash
ros2 param dump /review_publisher
```

它只打印 YAML。要保存到文件：

```bash
ros2 param dump /review_publisher > ~/review_saved.yaml
cat ~/review_saved.yaml
```

`>` 将输出写入文件，同名文件存在时会覆盖。本例用独立保存文件，不覆盖包内配置。

可以加载到运行中的节点：

```bash
ros2 param load /review_publisher ~/review_saved.yaml
```

需要检查每项返回结果，系统只读参数可能不接受运行时加载。

### 21.2 改安装的 YAML 为什么还要构建

直接 `--params-file 源码路径` 读取文件：保存后下一次启动就能读取，不必为 YAML 编译 C++。

本例 Launch 读取包安装目录的配置副本：改源码目录中的 YAML 后，要执行构建来更新安装副本，再重新启动。

本手册没有启用 `--symlink-install`。以后使用符号链接安装时，资源更新流程可能不同；不要把本例的拷贝安装规则套到所有项目上。

| 修改内容 | 本手册应做什么 |
| --- | --- |
| 改 C++ 逻辑 | 保存、构建、重启改过的程序 |
| 改运行时参数 | param set；本例不构建、不重启 |
| 改直接读取的外部 YAML | 保存，下一次启动明确加载 |
| 改包内安装的 YAML / Launch | 保存、构建更新安装副本、重新启动 |
| 改 msg / srv 字段结构 | 生成接口、构建依赖它的程序并重启相关节点 |

## 22. 动作：本次复习要掌握到什么程度

第一个模拟小项目暂不要求自己写 C++ 动作端点，但前四章中的概念要会选用。

动作接口 `.action` 的定义顺序是：

```text
目标 Goal 的字段
---
结果 Result 的字段
---
反馈 Feedback 的字段
```

执行时通常先发目标，过程中收反馈，最后取结果。取消是请求，由服务端处理，不能保证任何动作都能任意时刻立即停止。目标被接受也不等于最终成功。

单独复习海龟时，先停止当前 Launch，再开一个已加载 Jazzy 基础环境的终端：

```bash
ros2 run turtlesim turtlesim_node
```

另一终端：

```bash
source /opt/ros/jazzy/setup.bash
ros2 action list -t
ros2 action info /turtle1/rotate_absolute
ros2 interface show turtlesim/action/RotateAbsolute
ros2 action send_goal /turtle1/rotate_absolute turtlesim/action/RotateAbsolute "{theta: 1.57}" --feedback
```

`theta` 是绝对朝向，不是每次再转同样大小。要体验键盘取消，再启动 `ros2 run turtlesim turtle_teleop_key`，旋转字母键发动作目标，`F` 请求取消。普通箭头键发的是速度话题，机制不同。

Jazzy turtlesim 的 `delta` 实现使用归一化的“起始朝向减当前朝向”，不要直接按正负判断成功或方向；最终是否成功看状态。原第四章的简略角位移描述应据此补充。

通过标准：能解释为什么“导航到教学楼，报告进度，允许取消”适合动作，而“持续发布速度”适合话题。

## 23. 一套排错顺序

### 23.1 先检查程序是否真的运行

```bash
ros2 node list
ros2 node info /review_publisher
```

构建成功不等于程序已经启动。客户端短暂执行后退出，事后不在列表是正常情况。

### 23.2 再检查端点、类型、QoS

```bash
ros2 topic list -t
ros2 topic info /review/vehicle_status --verbose
ros2 interface show review_interfaces/msg/VehicleStatus
```

双方都启动却没有对应端点时，先查名称、重映射、类型及通信环境。名字和类型相同仍需兼容 QoS。

### 23.3 再看实际数据

```bash
ros2 topic echo --once /review/vehicle_status
```

日志写着 Published 不等于已经发了业务消息，要确认代码确实执行 publish，再看接收的数据。

### 23.4 参数不生效时

```bash
ros2 param get /review_publisher speed
```

参数值没变：查命令返回、节点名、参数名、类型和文件匹配。

参数已变但消息没变：查是否运行了新程序、回调是否读取当前参数、是否仍给消息写死数值、是否有旧发布者混入。

### 23.5 常见报错表

| 现象 | 优先检查 |
| --- | --- |
| `ros2: command not found` | 基础环境是否加载 |
| `Package ... not found` | 构建是否成功、当前终端加载的是不是新工作空间 |
| `No executable found` | 程序目标名和 install(TARGETS) 是否正确 |
| 找不到接口头文件 | 接口包是否生成，文件名大小写及程序依赖是否正确 |
| 编译错误很长 | 从自己源码的第一处实际 error 找起，不只看最后 Summary |
| 文件保存失败 | 父目录存在吗，文件路径正确吗 |
| 运行的还是旧行为 | 文件是否保存、构建是否成功、旧进程是否停止、加载了哪个工作空间 |
| 同一消息出现不同速度 | 是否有多个旧发布者 |
| YAML 没起作用 | 节点键、缩进、双下划线、参数类型、安装副本是否正确 |
| 服务一直等待 | 服务端是否运行、名称和类型是否匹配 |
| Launch 报 Python 错误 | 英文标点、缩进、括号、逗号及依赖 |

不要因为报错就删除整个工作空间或重装 ROS。先保留文件和错误信息，按目标逐项检查。

## 24. 前十章基础命令总表

表中的“包名、节点名”等是说明，执行时换成实际内容。

| 目标 | 命令通式或例子 |
| --- | --- |
| 启动程序 | `ros2 run 包名 程序名` |
| 看包内程序 | `ros2 pkg executables review_nodes` |
| 看节点 | `ros2 node list` |
| 看节点接口 | `ros2 node info /review_publisher` |
| 看话题及类型 | `ros2 topic list -t` |
| 看端点 | `ros2 topic info /review/vehicle_status --verbose` |
| 看接口字段 | `ros2 interface show review_interfaces/msg/VehicleStatus` |
| 取一条消息 | `ros2 topic echo --once /review/vehicle_status` |
| 观察接收频率 | `ros2 topic hz /review/vehicle_status` |
| 看服务及类型 | `ros2 service list -t` |
| 服务类型 | `ros2 service type /review/add_two_ints` |
| 按类型找服务 | `ros2 service find review_interfaces/srv/AddTwoInts` |
| 调服务 | `ros2 service call 服务名 类型 "请求数据"` |
| 看动作及类型 | `ros2 action list -t` |
| 看动作端点 | `ros2 action info 动作名` |
| 发动作目标 | `ros2 action send_goal 名称 类型 "目标" --feedback` |
| 看参数列表 | `ros2 param list /review_publisher` |
| 读参数 | `ros2 param get /review_publisher speed` |
| 改参数 | `ros2 param set /review_publisher speed 12.0` |
| 导出参数 | `ros2 param dump /review_publisher > ~/review_saved.yaml` |
| 加载运行参数 | `ros2 param load /review_publisher ~/review_saved.yaml` |
| 启动文件 | `ros2 launch review_nodes review.launch.py` |
| 构建及依赖 | `colcon build --packages-up-to review_nodes` |
| 帮助 | `ros2 topic --help`、`ros2 param set --help` 等 |

命令行也可以充当发布者。下面命令仅在已停止本例 C++ 发布者、并运行本例订阅者时试一次：

```bash
ros2 topic pub --once /review/vehicle_status review_interfaces/msg/VehicleStatus "{vehicle_name: manual_car, speed: 2.0, stopped: false}"
```

末尾实际数据使用 YAML 形式的英文标点。按需发布的值和 C++ 默认参数没有自动关联。

## 25. 代码阅读验收：不操作先讲清楚

复习后，先不看解释，回答：

1. 哪句决定程序目标名，哪句决定节点名？
2. 哪句创建发布接口，哪句真正发送？
3. lambda 是在定义时还是被调用时执行函数体？
4. timer 和 subscription 各由什么事件触发？
5. 为什么订阅端用 `->`，发布端填写消息时用 `.`？
6. 接口定义中的 sum 为什么不会自己算出来？
7. wait_for_service 与等待 future 分别等什么？
8. 修改运行参数为什么不必改 `.msg`？
9. 为什么 YAML 顶层写节点名？
10. 为什么 Launch 不保证第一个程序已经就绪？

答不出来时回到对应节，不必从头重读全部内容。

## 26. 实操验收：逐步减少提示

### 第一遍：按手册完成

允许复制代码，但每个文件要知道放在哪、由谁使用、怎样验证。记录报错和解决方法。

### 第二遍：只看目标，不看操作段

利用已经建好的练习包，自己完成：

- 启动车辆发布、订阅并查消息结构。
- 修改速度，检查真实话题消息。
- 发一次求和请求，检查对应响应。
- 修改 YAML 车辆名并让下一次启动生效。
- 说明哪些步骤需要构建，哪些只需运行参数命令。

可以查命令语法，不要求盲打全部参数。目标是你自己能决定下一步做什么。

### 第三遍：故障解释

不用破坏系统，口头或在笔记中分析：

- 只改一端话题名为什么收不到？
- 改了 C++ 不构建为什么仍是旧结果？
- 参数已改，但程序用启动时缓存为什么输出不变？
- 改源码 YAML，却读取安装副本为什么可能仍是旧值？
- 客户端退出、服务端不退出为什么正常？

这三遍完成，就有条件开始独立小项目，不必等所有代码都能背下来。

## 27. 独立小项目任务书：车辆状态监控与控制

这是下一阶段任务，不是现在继续复制本手册的代码。建议另建 `~/vehicle_project_ws`，使用自己取的包名与接口名。

### 最小功能

1. 一个车辆模拟节点，保存当前模拟速度和车辆名称。
2. 定时发布自定义车辆状态，至少包括名称、速度、停车状态。
3. 一个监控节点，订阅状态并判断是否超过速度阈值。
4. 一个设置速度的服务：请求包含目标速度，响应包含是否成功和说明文字。
5. 初始速度和超速阈值由参数配置，能从 YAML 加载。
6. 一个 Launch 启动整个系统。

这里设置速度服务的结构与本手册的求和服务不同。你要根据任务自行设计 `.srv`，不能只把求和变量改名就认为业务完成。

### 重要的设计选择

模拟节点的当前速度只选一个清晰的数据来源。例如：启动参数初始化内部状态，设置速度服务更新这个状态，发布回调读取这个状态。

不要一边每次回调强制从参数覆盖速度，一边服务改另一个内部变量，否则服务设置可能下一秒又被旧参数盖回去。是否支持运行时调速参数，应明确设计，第一版不必同时支持两条设置路径。

### 最小验收场景

| 操作 | 你应能验证的结果 |
| --- | --- |
| 一条 Launch 命令启动 | 所有需要的节点都出现 |
| 设置正常速度 | 话题中的速度更新，监控没有超速提示 |
| 设置超过阈值的速度 | 监控给出清晰超速提示 |
| 设置 0.0 | 消息停车状态为真 |
| 请求不允许的速度 | 服务返回失败及原因，状态保持原值 |
| 关闭后重新启动 | 按已设计的启动配置初始化 |

速度范围、单位、是否接受负数等由你在第一版需求中明确。项目先用模拟数据，不接真实车辆。

### 开始写代码之前先交这四样

- 包和文件清单。
- 节点职责表。
- 话题/服务名称及类型、字段定义。
- 从创建目录到启动验证的操作计划。

我先检查你的设计和接口，再由你自己写；遇到问题按报错定位，尽量不直接替你写完全部项目。

## 28. 笔记仓库检查中发现的修订

检查基准：[ROS2study main](https://github.com/matianhong1/ROS2study)，提交 `a4f2de573d6f788f3ab30879f4a39ca3788d281a`。十章和两份辅助笔记都存在，主线够用，但以下旧内容要留意：

1. 第六章 6.6 仍保留 `[node, publisher, count = 0]() mutable`，与你课堂曾遇到的 Jazzy rclcpp 回调推导错误相对应。应换成外部 `int count = 0;`，回调捕获 `[node, publisher, &count]`，并同步修改 6.6.2 解释；引用对象必须在回调执行期间有效。本手册没有沿用该旧 mutable 写法。
2. 第二章 `topic hz` 应解释成该工具的实际接收频率，不直接等同发布端时钟频率。
3. 第四章和前四章总结应补充 turtlesim `delta` 的实际符号定义，不能简单按“终点减起点”或“累计转角”理解。
4. 第四章 4.6.3 要演示新动作替换旧动作，应明确按旋转字母键；普通方向键发送速度消息中止动作是另一种路径。

仓库目前以 Markdown 为主，还没有 README 总入口和独立示例源码目录。建议以后把自己实际运行成功的源码包备份进去；不必为了复习先重新组织全部仓库。代码别只留在虚拟机里。

## 29. 备份与 Git 的边界

Windows 的 ROS2study 仓库只会上传那个目录下已加入 Git 的文件，不会自动发现 Ubuntu 主目录里的程序。

以后备份项目，至少保留两个包的源码与配置：`.cpp`、`.msg`、`.srv`、CMakeLists.txt、package.xml、launch、config。通常不提交自动生成的 build、install、log。

本手册不要求你现在在 Ubuntu 重新配置 Git。先完成练习，再选择在 Ubuntu 直接使用仓库，或把源码明确复制到 Windows 仓库中备份。无论采用哪条路线，都应确认被提交的确实包含源文件。

## 30. 参考与验证范围

- [ROS 2 Jazzy 官方教程目录](https://docs.ros.org/en/jazzy/Tutorials.html)
- [自定义接口](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Custom-ROS2-Interfaces.html)
- [C++ 发布订阅](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Cpp-Publisher-And-Subscriber.html)
- [C++ 服务](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Cpp-Service-And-Client.html)
- [参数](https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Parameters/Understanding-ROS2-Parameters.html)
- [Launch](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Launch/Creating-Launch-Files.html)
- [Jazzy rclcpp 回调类型推导源码](https://github.com/ros2/rclcpp/blob/jazzy/rclcpp/include/rclcpp/function_traits.hpp)
- [Jazzy 海龟动作实际实现](https://github.com/ros/ros_tutorials/blob/jazzy/turtlesim/src/turtle.cpp)

本手册按已学、已在课堂验证过的机制重新组织了独立练习。Python、XML、YAML 及 shell 命令块已通过语法检查，并核对了四个完整 C++ 程序与四个构建目标的对应关系；没有完成 ROS C++ 编译和虚拟机端到端运行，所以不能把静态检查当成实机通过。实际结果以你逐步操作时的构建输出和通信数据为准。
