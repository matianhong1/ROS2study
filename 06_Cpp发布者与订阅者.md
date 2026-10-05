# 第6章 C++ 发布者与订阅者

> 环境：Ubuntu 24.04 + ROS 2 Jazzy  
> 前置：第2章的话题与消息、第5章的构建运行流程  
> 学习方法：先解释每一步为什么做，再分步操作；不要求一次理解全部代码。

## 6.1 本章要做什么

编写两个程序：发布者每秒发送一次模拟车速，订阅者收到车速后打印。车速由代码生成，不是真实车辆测量值。

| 对象 | 本章约定 |
| --- | --- |
| 工作空间 | `~/ros2_ws`，继续使用第5章的工作空间 |
| 新功能包 | `cpp_pubsub` |
| 发布者源文件／程序名／节点名 | `speed_pub.cpp`／`speed_pub`／`/speed_publisher` |
| 订阅者源文件／程序名／节点名 | `speed_sub.cpp`／`speed_sub`／`/speed_subscriber` |
| 话题 | `/vehicle/speed` |
| 消息类型 | `std_msgs/msg/Float64` |
| 消息字段 | `data`，本练习约定单位为 m/s |

发布者节点通过 `/vehicle/speed` 发送消息；订阅者节点订阅同一话题、使用同一类型并处理消息。话题是命名通道，不是一个独立运行的中转程序。

学习目标：理解创建发布者、创建订阅者、消息对象、定时器、回调与 spin，并能构建运行两个自己的通信节点。

## 6.2 从第五章的节点出发

第五章的节点只输出一次日志并等待。本章增加业务接口：

| 已学过的部分 | 本章新增部分 |
| --- | --- |
| 初始化、创建节点、日志、spin、关闭 | 创建发布者和订阅者 |
| CMake 构建规则、colcon、source、run | 定时触发发送、收到消息后执行处理代码 |

发布不是打印。`RCLCPP_INFO` 是记录日志；`publisher->publish(message)` 才是发布业务话题消息。即使删除发送端的打印语句，消息仍可以被接收。

## 6.3 消息类型与 C++ 写法

查看接口的命令：

```bash
ros2 interface show std_msgs/msg/Float64
```

结构是：

```text
float64 data
```

| 场合 | 写法 |
| --- | --- |
| 命令行中的类型 | `std_msgs/msg/Float64` |
| C++ 中的类型 | `std_msgs::msg::Float64` |
| C++ 头文件 | `"std_msgs/msg/float64.hpp"` |

`std_msgs` 是标准消息功能包，`msg` 是接口类别，`Float64` 表示具有一个双精度浮点数据字段的消息。本类型不自带速度单位，m/s 是我们这个练习的约定。真实系统应根据需求选择含时间戳、坐标系等信息的接口。

创建并填写消息：

```cpp
std_msgs::msg::Float64 message;
message.data = 5.0;
```

`message` 是对象，所以使用 `.` 访问字段。它是一个包含 data 字段的消息对象，不只是一个裸 double。

## 6.4 新概念：回调与定时器

**回调函数是交给框架、在指定事件发生时调用的处理代码。**

| 事件 | 对应回调做什么 |
| --- | --- |
| 定时器到期 | 生成并发布一条速度消息 |
| 订阅者收到消息 | 读取 data 并打印 |

定时器设置周期，本章为1秒。`spin` 让执行器等待并处理这些事件；它不会不断重跑整个 main，也不会重复执行它前面的所有语句。

本章用 lambda 写短回调。例如：

```cpp
auto callback = []() {
  // 事件发生时执行这里
};
```

| 部分 | 含义 |
| --- | --- |
| `[]` | 捕获列表，说明要使用哪些外部变量 |
| `()` | 回调的参数 |
| `{...}` | 回调函数体 |
| `auto callback = ...` | 把这个可调用对象保存到变量 |

定义回调不等于立即执行回调。课堂会单独拆解 lambda，不需要现在背下所有语法。

## 6.5 环境与创建新包

本章不需要海龟。第五章的 hello_node 可以停止，原功能包保留。

构建终端执行：

```bash
source /opt/ros/jazzy/setup.bash
cd ~/ros2_ws/src
ros2 pkg create cpp_pubsub --build-type ament_cmake --license Apache-2.0 --dependencies rclcpp std_msgs
```

依赖 `rclcpp` 用于 ROS 2 C++ 通信，依赖 `std_msgs` 用于消息类型。新包位于 `~/ros2_ws/src/cpp_pubsub`，与 `my_first_pkg` 并列。若同名包已经存在，先检查，不重复创建或删除。

## 6.6 发布者完整代码

编辑：

```bash
nano ~/ros2_ws/src/cpp_pubsub/src/speed_pub.cpp
```

```cpp
#include <chrono>
#include <memory>
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/float64.hpp"

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  auto node = std::make_shared<rclcpp::Node>("speed_publisher");

  auto publisher = node->create_publisher<std_msgs::msg::Float64>(
    "/vehicle/speed", 10);

  auto publish_speed = [node, publisher, count = 0]() mutable {
    std_msgs::msg::Float64 message;
    message.data = 5.0 + 0.5 * count;
    publisher->publish(message);
    RCLCPP_INFO(node->get_logger(), "Publishing speed: %.1f m/s", message.data);
    ++count;
  };

  auto timer = node->create_wall_timer(std::chrono::seconds(1), publish_speed);

  rclcpp::spin(node);
  rclcpp::shutdown();
  return 0;
}
```

保存：`Ctrl+O`、Enter；退出：`Ctrl+X`。nano 是编辑器，不负责编译。

### 6.6.1 创建发布者

```cpp
node->create_publisher<std_msgs::msg::Float64>("/vehicle/speed", 10);
```

| 部分 | 含义 |
| --- | --- |
| `node->` | 在这个节点对象上调用成员函数 |
| `create_publisher` | 创建发布者接口 |
| `<std_msgs::msg::Float64>` | 指定消息类型 |
| `"/vehicle/speed"` | 指定话题名称 |
| `10` | QoS 中保留历史的深度设置，不是10 Hz、节点数或发布次数 |

本章双方采用兼容的默认 QoS 设置，具体策略以后学习。创建发布者只是准备接口；每次调用 publish 才发送消息。

### 6.6.2 定时发布回调

`[node, publisher, count = 0]` 将智能指针捕获到回调中，并初始化回调自己的计数值。`mutable` 允许回调修改按值捕获的 count。

每次回调执行：创建消息 → 写入 data → 发布 → 打印 → count 加1。预期车速依次为 `5.0、5.5、6.0…`。

`std::chrono::seconds(1)` 表示1秒时长；`create_wall_timer` 创建周期定时器。实际执行时间可能受系统调度影响，不保证毫秒级精确。

保留 `timer` 变量，使定时器在 spin 期间保持存在。这里没有手写 while；重复执行的是定时回调。

## 6.7 订阅者完整代码

编辑：

```bash
nano ~/ros2_ws/src/cpp_pubsub/src/speed_sub.cpp
```

```cpp
#include <memory>
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/float64.hpp"

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);
  auto node = std::make_shared<rclcpp::Node>("speed_subscriber");

  auto receive_speed = [node](std_msgs::msg::Float64::ConstSharedPtr message) {
    RCLCPP_INFO(node->get_logger(), "Received speed: %.1f m/s", message->data);
  };

  auto subscription = node->create_subscription<std_msgs::msg::Float64>(
    "/vehicle/speed", 10, receive_speed);

  rclcpp::spin(node);
  rclcpp::shutdown();
  return 0;
}
```

### 6.7.1 创建订阅者

`create_subscription<类型>(话题名称, 历史深度, 接收回调)` 指定监听哪个话题、接收什么结构，以及收到数据之后做什么。

`receive_speed` 不需要自己反复调用，收到消息且执行器处理该事件时，它会被调用。没有新消息时，不会凭空不断打印。

### 6.7.2 为什么这里用箭头

`ConstSharedPtr` 是指向只读消息的共享智能指针，因此通过 `message->data` 读取内容。

| 发布端 | 接收端 |
| --- | --- |
| `message` 是消息对象 | `message` 是指向消息的智能指针 |
| `message.data` | `message->data` |

两个 message 是不同程序中的局部变量。跨进程通信传输消息内容，不能理解成把发布端的 C++ 指针直接交给订阅端。

保留 `subscription`，使订阅接口在 spin 期间保持存在。

## 6.8 设置构建说明

仅对本章新建的 `cpp_pubsub` 包操作，保留第五章的 `my_first_pkg` 不变。

```bash
nano ~/ros2_ws/src/cpp_pubsub/CMakeLists.txt
```

将新包的 CMakeLists.txt 整体替换为下列完整内容，避免重复添加目标：

```cmake
cmake_minimum_required(VERSION 3.8)
project(cpp_pubsub)

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)

add_executable(speed_pub src/speed_pub.cpp)
ament_target_dependencies(speed_pub rclcpp std_msgs)
target_compile_features(speed_pub PUBLIC cxx_std_17)

add_executable(speed_sub src/speed_sub.cpp)
ament_target_dependencies(speed_sub rclcpp std_msgs)
target_compile_features(speed_sub PUBLIC cxx_std_17)

install(TARGETS
  speed_pub
  speed_sub
  DESTINATION lib/${PROJECT_NAME}
)

ament_package()
```

规则含义与第五章一致，这次定义两个程序，都依赖 rclcpp 和 std_msgs，并明确要求 C++17。创建包时已声明这两个依赖，通常不需要再修改 package.xml。

## 6.9 编译与运行

构建终端：加载基础 ROS 2 环境，不加载本工作空间 overlay；在根目录构建。

```bash
cd ~/ros2_ws
colcon build --packages-select cpp_pubsub
```

Summary 成功完成一个包，即使这个包包含两个可执行程序，包数量仍是1。

新终端1，发布端：

```bash
source /opt/ros/jazzy/setup.bash
source ~/ros2_ws/install/local_setup.bash
ros2 run cpp_pubsub speed_pub
```

新终端2，接收端：

```bash
source /opt/ros/jazzy/setup.bash
source ~/ros2_ws/install/local_setup.bash
ros2 run cpp_pubsub speed_sub
```

预期发送端每秒打印 Publishing，接收端每次收到后打印 Received。启动订阅者之前的消息通常不会补发，因为本练习采用默认 volatile 持久性设置；不要要求从5.0开始接收。

source 是加载环境，不是编译或运行。修改 C++ 代码后需重新构建、停止旧程序，再启动新程序。

## 6.10 用第二章命令验证通信

第三个查询终端：

```bash
ros2 node list
ros2 node info /speed_publisher
ros2 node info /speed_subscriber
ros2 topic list -t
ros2 topic info /vehicle/speed
ros2 topic echo /vehicle/speed
```

只运行这两个业务节点且没有其他工具订阅本话题时，info 预期为1个发布端、1个订阅端。运行 echo 后，订阅端数量会增加，因为它也会临时订阅。

停止 echo 后可以测量：

```bash
ros2 topic hz /vehicle/speed
```

预期测得接收频率接近1 Hz，虚拟机调度等因素可造成偏差。也可用 rqt_graph 观察节点与话题的关系。

## 6.11 理解性练习

1. 停止订阅者：发布者仍继续发送，话题不要求等待接收端答复。
2. 重启订阅者：它接收新到来的消息。
3. 停止发布者：订阅者仍在等待，但没有新的速度日志。
4. 把发布周期改为 `std::chrono::milliseconds(500)`，重新构建运行，预期频率约2 Hz。
5. 只把接收端话题名改为 `/vehicle/other_speed`，重新构建运行，观察为什么收不到；随后改回原名。

每次只改变一个因素。两个接口名称一致、类型一致，还需要 QoS兼容、网络与发现环境正常，才具备通信条件。

## 6.12 常见混淆

| 混淆 | 正确理解 |
| --- | --- |
| 打印日志就是发布速度 | 日志与业务消息是不同接口；publish 才发布速度 |
| 10 表示每秒发布10次 | 10 是历史深度；定时周期决定本例发送频率 |
| spin 重复运行 main | spin 处理事件，调用就绪回调 |
| 订阅者要指定发布节点名字 | 本例根据话题名称和类型订阅，无需指定发布节点名 |
| 一个包只能有一个节点 | 本包安装两个程序，分别运行可创建两个节点 |
| 回调定义时就发布数据 | 定义回调只是准备可调用对象，事件触发后才执行 |
| 收到的消息就是发送端变量的地址 | 本例跨进程传消息内容，不共享该局部指针 |

报错时检查当前目录、文件名、CMake 依赖和目标、是否构建成功、是否加载环境。消息名称和类型以本机查询结果为准。

## 6.13 本章验收

- [ ] 能区分包名、程序名、节点名、话题名、类型和实际消息。
- [ ] 能解释 create_publisher 与 publish 的不同。
- [ ] 能解释定时回调和接收回调分别何时执行。
- [ ] 能解释 timer 周期、spin 和历史深度10的不同作用。
- [ ] 能读懂 message.data 与 message->data 的区别。
- [ ] 能构建并运行两个程序，验证发布订阅通信。
- [ ] 能解释单独停止一端时另一端的表现。

不要求立即默写所有 C++ 模板与 lambda；先能解释、参考笔记运行，再通过练习熟悉。

## 6.14 参考与验证说明

- [ROS 2 Jazzy 官方：C++ 发布者与订阅者](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Cpp-Publisher-And-Subscriber.html)

本笔记使用 main 内创建节点及 lambda 回调，延续第五章写法。官方教程常用继承 Node 的类组织程序，两者采用同样的发布订阅机制。先学习本例，再学习类封装，避免同时引入过多语法。

本文为课程代码与教学步骤，尚未在你的虚拟机中验证；当前笔记生成环境没有安装 ROS 2，因此不声称本例已通过本地编译。课堂将依据你的构建输出和通信结果完成验证。
