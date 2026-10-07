# 第七章：C++ 服务端与客户端

> 环境：Ubuntu 24.04、ROS 2 Jazzy、C++17。
> 前置知识：第五章的功能包与构建、第六章的节点与回调函数、第三章的服务命令。
> 学习方式：先理解，再逐步实操。完整代码供保存和复习，不要求一次全部看懂。

## 7.1 这一章要做什么

上一章，你写了发布者和订阅者，通过 `/vehicle/speed` 话题发送速度。

这一章，你将写两个程序：一个提出计算请求，一个处理请求并返回答案。

具体例子：客户端发送 `a = 2`、`b = 3`，服务端计算 `a + b`，返回 `sum = 5`。

为什么先用加法？因为它的计算逻辑简单，可以把注意力集中在通信过程上。理解后，同样的结构可以用来请求重置仿真、查询状态或设置设备。

学完后，你应能解释：

- 谁是客户端，谁是服务端。
- 服务名称和服务类型分别是什么。
- 请求与响应的数据结构怎样区分。
- `create_service`、`create_client` 和 `async_send_request` 各做什么。
- 为什么程序需要处理回调或等待响应。

## 7.2 先复习服务

第三章用过：

```bash
ros2 service call /spawn turtlesim/srv/Spawn "{x: 2.0, y: 2.0, theta: 0.0, name: ''}"
```

当时，命令行工具充当客户端，海龟节点提供服务端接口。你给出请求，海龟节点处理后返回响应。

本章把“命令行发送请求”换成“自己写的 C++ 程序发送请求”。

| 对比 | 话题通信 | 服务通信 |
| --- | --- | --- |
| 角色 | 发布者、订阅者 | 客户端、服务端 |
| 数据组织 | 按消息类型发送消息 | 按服务类型组织请求和响应 |
| 使用方式 | 发布方可以不断发送数据 | 客户端发起请求，服务端返回响应 |
| 本课程例子 | 发送车辆速度 | 请求计算两个整数的和 |

服务不会因为“有返回值”就自动具备动作的进度反馈与取消协议。耗时任务的进度和取消仍然要考虑动作。

一个节点可以拥有多个通信接口，也可以同时是客户端和服务端。这些名称描述的是接口角色，不是限制节点只能做一种事。

## 7.3 本章所有名称的对应关系

| 类别 | 本章名称 | 含义 |
| --- | --- | --- |
| 功能包 | `cpp_srvcli` | 保存源码和构建配置 |
| 服务端源码 | `add_server.cpp` | 服务端的 C++ 源文件 |
| 客户端源码 | `add_client.cpp` | 客户端的 C++ 源文件 |
| 服务端可执行程序 | `add_server` | `ros2 run` 启动的程序 |
| 客户端可执行程序 | `add_client` | `ros2 run` 启动的程序 |
| 服务端节点 | `/addition_server` | 运行时的节点名称 |
| 客户端节点 | `/addition_client` | 运行时的节点名称 |
| 服务名称 | `/add_two_ints` | 客户端寻找的服务接口名称 |
| 服务类型 | `example_interfaces/srv/AddTwoInts` | 请求和响应的数据模板 |

不要把包名、程序名、节点名、服务名混在一起。它们可以取不同名字。本章故意让节点名和程序名不同，便于区分。

## 7.4 请求和响应是什么

查看接口：

```bash
ros2 interface show example_interfaces/srv/AddTwoInts
```

核心内容：

```text
int64 a
int64 b
---
int64 sum
```

- `---` 上面是请求：客户端填写 `a` 和 `b`。
- `---` 下面是响应：服务端填写 `sum`。
- `int64` 是有符号 64 位整数。

你可以继续用“结构体模板”理解：服务接口生成了请求类型和响应类型，而程序创建的是这些类型的具体对象。

| 表达方式 | 位置 | 含义 |
| --- | --- | --- |
| `example_interfaces/srv/AddTwoInts` | ROS 命令 | 服务类型名称 |
| `example_interfaces::srv::AddTwoInts` | C++ | 服务类型 |
| `AddTwoInts::Request` | C++ | 请求类型 |
| `AddTwoInts::Response` | C++ | 响应类型 |

这里 `example_interfaces` 是提供示例接口的功能包；`srv` 表示服务接口；`AddTwoInts` 表示两个整数求和。

## 7.5 准备环境和创建功能包

启动已有的 Ubuntu 虚拟机，登录后按 `Ctrl + Alt + T` 打开终端。

先停止之前的练习程序，避免多个窗口混淆。下面操作都在 Ubuntu 终端中执行。

```bash
source /opt/ros/jazzy/setup.bash
cd ~/ros2_ws/src
ros2 pkg create cpp_srvcli --build-type ament_cmake --license Apache-2.0 --dependencies rclcpp example_interfaces
```

解释：

- `source`：给当前终端加载 ROS 环境。
- `cd`：进入工作空间的 `src` 文件夹。
- `ros2 pkg create`：生成新功能包。
- `rclcpp`：编写 C++ ROS 2 节点要用的库。
- `example_interfaces`：本章使用的服务类型所在的包。

如果提示文件夹已存在，先检查已有内容，不要直接删包重新创建。

## 7.6 完整服务端代码

打开文件：

```bash
nano ~/ros2_ws/src/cpp_srvcli/src/add_server.cpp
```

输入下面的完整代码：

```cpp
#include <memory>

#include "rclcpp/rclcpp.hpp"
#include "example_interfaces/srv/add_two_ints.hpp"

// 为较长的类型起一个短名字。
using AddTwoInts = example_interfaces::srv::AddTwoInts;

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);

  auto node = std::make_shared<rclcpp::Node>("addition_server");

  // 请求到来后，ROS 2 调用这个函数。
  auto handle_request = [node](
    std::shared_ptr<AddTwoInts::Request> request,
    std::shared_ptr<AddTwoInts::Response> response)
  {
    response->sum = request->a + request->b;

    RCLCPP_INFO(
      node->get_logger(),
      "Request: a=%lld, b=%lld; response: sum=%lld",
      static_cast<long long>(request->a),
      static_cast<long long>(request->b),
      static_cast<long long>(response->sum));
  };

  auto service = node->create_service<AddTwoInts>(
    "/add_two_ints", handle_request);

  RCLCPP_INFO(node->get_logger(), "Service is ready: /add_two_ints");

  rclcpp::spin(node);
  rclcpp::shutdown();
  return 0;
}
```

保存：`Ctrl + O`，按 Enter 确认文件名；退出：`Ctrl + X`。

### 7.6.1 `using` 是什么

```cpp
using AddTwoInts = example_interfaces::srv::AddTwoInts;
```

给已有类型起一个别名。以后写 `AddTwoInts`，就是指右边那个类型。它不是创建对象，也不是创建新的服务。

### 7.6.2 回调函数收到了什么

```cpp
std::shared_ptr<AddTwoInts::Request> request
std::shared_ptr<AddTwoInts::Response> response
```

这两个参数都是智能指针：

- `request` 指向收到的请求对象。
- `response` 指向要填写的响应对象。

因为通过指针访问成员，所以使用 `->`。

```cpp
response->sum = request->a + request->b;
```

这句取出请求里的两个整数，计算后填入响应。回调结束后，ROS 2 将响应发给客户端。

`RCLCPP_INFO` 只是日志。删除日志，不会删除计算和响应的功能。

### 7.6.3 创建服务接口

```cpp
auto service = node->create_service<AddTwoInts>(
  "/add_two_ints", handle_request);
```

- `<AddTwoInts>`：服务类型。
- `"/add_two_ints"`：服务名称。
- `handle_request`：收到请求时执行的回调函数。
- `service`：保存服务接口对象的智能指针，使接口在需要期间保持存在。

创建接口不等于直接执行加法。请求到达并被执行器处理时，回调才会执行。

### 7.6.4 为什么 `spin` 很重要

```cpp
rclcpp::spin(node);
```

让执行器等待并处理节点的工作，其中包括服务请求回调。程序在这里持续运行，通常按 `Ctrl + C` 后继续清理并退出。

## 7.7 完整客户端代码

为了先学通信，本章先把请求的数字写在代码里，不加入命令行参数解析。

```bash
nano ~/ros2_ws/src/cpp_srvcli/src/add_client.cpp
```

完整代码：

```cpp
#include <chrono>
#include <memory>

#include "rclcpp/rclcpp.hpp"
#include "example_interfaces/srv/add_two_ints.hpp"

using AddTwoInts = example_interfaces::srv::AddTwoInts;

int main(int argc, char * argv[])
{
  rclcpp::init(argc, argv);

  auto node = std::make_shared<rclcpp::Node>("addition_client");
  auto client = node->create_client<AddTwoInts>("/add_two_ints");

  // 每次最多等待一秒；未找到服务则继续等待。
  while (!client->wait_for_service(std::chrono::seconds(1))) {
    if (!rclcpp::ok()) {
      RCLCPP_INFO(node->get_logger(), "Waiting interrupted. Exiting.");
      rclcpp::shutdown();
      return 0;
    }
    RCLCPP_INFO(node->get_logger(), "Service unavailable, waiting...");
  }

  auto request = std::make_shared<AddTwoInts::Request>();
  request->a = 2;
  request->b = 3;

  RCLCPP_INFO(node->get_logger(), "Sending request: a=2, b=3");
  auto future = client->async_send_request(request);

  // 处理节点的工作，直到收到响应、超时或被中断。
  auto status = rclcpp::spin_until_future_complete(
    node, future, std::chrono::seconds(10));

  if (status == rclcpp::FutureReturnCode::SUCCESS) {
    auto response = future.get();
    RCLCPP_INFO(
      node->get_logger(), "Received sum: %lld",
      static_cast<long long>(response->sum));
  } else {
    RCLCPP_ERROR(node->get_logger(), "No response: timeout or interruption.");
    client->remove_pending_request(future);
    rclcpp::shutdown();
    return 1;
  }

  rclcpp::shutdown();
  return 0;
}
```

### 7.7.1 创建客户端接口

```cpp
auto client = node->create_client<AddTwoInts>("/add_two_ints");
```

创建客户端接口，指定要联系的服务名称与类型。此时没有发送请求，也没有创建服务端。

### 7.7.2 等待服务

```cpp
client->wait_for_service(std::chrono::seconds(1))
```

等待服务可用，成功返回 `true`，一次等待到期仍不可用则返回 `false`。`!` 表示逻辑取反。

因此 `while (!...)` 表示：没找到服务就继续循环。

这里的一秒是单次等待时长，不是发布频率。等待期间按 `Ctrl + C`，`rclcpp::ok()` 可以让程序识别退出请求。

### 7.7.3 创建并填写请求

```cpp
auto request = std::make_shared<AddTwoInts::Request>();
request->a = 2;
request->b = 3;
```

先创建请求对象，再填写其字段。服务端稍后读取的就是这两个值。

### 7.7.4 发送请求和取得结果

```cpp
auto future = client->async_send_request(request);
```

发送请求，返回一个用于取得将来结果的对象。`future` 是变量名，可以理解为“等结果到来的凭据”，此时它还不是求和结果。

```cpp
rclcpp::spin_until_future_complete(node, future, std::chrono::seconds(10));
```

让执行器处理工作，并等待这个请求的响应。本例最多等待十秒。它与前面的“等待服务存在”是两个不同阶段。

成功后：

```cpp
auto response = future.get();
```

取出响应对象的智能指针，再用 `response->sum` 读取答案。

本例虽然采用异步发送 API，但随后主动等待结果，所以整体表现是发一次请求、等待一次响应、打印答案、退出。

超时分支中的 `remove_pending_request` 清理客户端保存的待处理请求记录，不代表取消了服务端已经执行的工作。

## 7.8 构建配置

打开：

```bash
nano ~/ros2_ws/src/cpp_srvcli/CMakeLists.txt
```

本章新建的 `cpp_srvcli` 可以使用下面的完整配置。不要把它覆盖到第五、六章的其他包里。

```cmake
cmake_minimum_required(VERSION 3.8)
project(cpp_srvcli)

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(example_interfaces REQUIRED)

add_executable(add_server src/add_server.cpp)
target_compile_features(add_server PUBLIC cxx_std_17)
ament_target_dependencies(add_server rclcpp example_interfaces)

add_executable(add_client src/add_client.cpp)
target_compile_features(add_client PUBLIC cxx_std_17)
ament_target_dependencies(add_client rclcpp example_interfaces)

install(TARGETS
  add_server
  add_client
  DESTINATION lib/${PROJECT_NAME}
)

ament_package()
```

回顾：`find_package` 找依赖；`add_executable` 定义程序及源码；`ament_target_dependencies` 配置目标依赖；`install` 指定安装位置。

建包命令已将 `rclcpp`、`example_interfaces` 添加到 `package.xml`。本章不需要手动再添加一次。

## 7.9 构建并运行服务端

```bash
cd ~/ros2_ws
colcon build --packages-select cpp_srvcli
source ~/ros2_ws/install/setup.bash
ros2 run cpp_srvcli add_server
```

预期最后看到：

```text
[addition_server]: Service is ready: /add_two_ints
```

时间戳等细节可能不同。服务端继续等待，不返回命令提示符，属于正常现象。

构建失败时先处理报错，不要接着运行；否则可能运行到旧版本程序。

## 7.10 先用熟悉的命令测试服务端

保留服务端运行，新开第二个终端：

```bash
source /opt/ros/jazzy/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 service list -t
ros2 service type /add_two_ints
ros2 interface show example_interfaces/srv/AddTwoInts
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 2, b: 3}"
```

响应中应包含 `sum=5`。服务端窗口应打印收到的两个数字和计算结果。

这个阶段只有我们写的服务端在运行，客户端由 `ros2 service call` 命令提供。

## 7.11 运行自己写的客户端

仍在第二个终端执行：

```bash
ros2 run cpp_srvcli add_client
```

预期输出：

```text
[addition_client]: Sending request: a=2, b=3
[addition_client]: Received sum: 5
```

客户端完成一次请求后退出，终端回到提示符；服务端继续运行。这是我们的程序设计，不是 ROS 2 强制客户端必须退出。

客户端很快结束，因此事后用 `ros2 node list` 可能只看到服务端。不要据此判断客户端没有运行过。

## 7.12 三个理解实验

### 实验一：修改请求

将客户端的 `request->a` 改为 `10`，`request->b` 改为 `7`，同时更新发送日志，避免日志与实际请求不一致。

修改源码后：

```bash
cd ~/ros2_ws
colcon build --packages-select cpp_srvcli
source ~/ros2_ws/install/setup.bash
ros2 run cpp_srvcli add_client
```

预期收到 `17`。服务端的求和代码没有改变，所以不需要为了这次客户端修改而重启仍在运行的服务端。

### 实验二：先启动客户端

1. 在服务端终端按 `Ctrl + C` 停止服务端。
2. 运行客户端，观察它等待服务。
3. 在另一终端重新启动服务端。
4. 观察客户端找到服务、发送请求并得到结果。

理解：创建客户端不代表服务端一定存在。本例加入等待逻辑，是为了应对启动顺序。

### 实验三：日志与通信

思考：如果只删掉服务端回调里的 `RCLCPP_INFO`，保留求和赋值，客户端还能收到答案吗？

答案：能。日志用于观察，响应字段的填写才是这个回调完成计算的关键。ROS 2 负责把响应传回去。

## 7.13 指令速查

| 用途 | 通式 | 本章例子 |
| --- | --- | --- |
| 列出服务和类型 | `ros2 service list -t` | 同左 |
| 查看服务类型 | `ros2 service type 服务名` | `ros2 service type /add_two_ints` |
| 查看接口字段 | `ros2 interface show 类型` | `ros2 interface show example_interfaces/srv/AddTwoInts` |
| 调用服务 | `ros2 service call 服务名 类型 "请求"` | `ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 2, b: 3}"` |
| 构建指定包 | `colcon build --packages-select 包名` | `colcon build --packages-select cpp_srvcli` |
| 加载工作空间 | `source 工作空间/install/setup.bash` | `source ~/ros2_ws/install/setup.bash` |
| 运行程序 | `ros2 run 包名 程序名` | `ros2 run cpp_srvcli add_server` |

## 7.14 常见问题

| 现象 | 先检查什么 |
| --- | --- |
| `Package 'cpp_srvcli' not found` | 是否构建成功，当前终端是否加载工作空间 |
| 找不到可执行程序 | CMake 是否定义目标、安装目标，构建是否成功 |
| 缺少接口头文件或依赖包 | 是否写对 `example_interfaces`，是否安装该包 |
| 客户端一直等待服务 | 服务端是否运行，名称和类型是否一致，两个终端是否同一 ROS 通信环境 |
| 改代码后仍输出旧值 | 是否保存、重新构建，并重启修改过的程序 |
| 服务端启动后不打印求和结果 | 是否真的发送过请求；它不会自动计算 |
| 客户端收到答案后消失 | 本例正常完成并退出 |

如果确认缺少接口依赖，可以安装：

```bash
sudo apt install ros-jazzy-example-interfaces
```

不需要为了开始本章就重复安装已经存在的包。

## 7.15 自检题与答案

先自己回答，再看下面的对应答案。

1. `/add_two_ints` 与 `example_interfaces/srv/AddTwoInts` 有什么区别？
2. 谁填写 `a`、`b`，谁填写 `sum`？
3. 创建服务接口时会立即执行加法吗？
4. `async_send_request` 返回的是答案本身吗？
5. 客户端退出后，服务端为什么还在运行？
6. 删除日志会不会让服务通信失效？

答案：

1. 前者是服务名称，后者是服务类型。
2. 客户端填写请求中的 `a`、`b`，服务端填写响应中的 `sum`。
3. 不会。请求到达并被处理时，回调才执行计算。
4. 不是。它返回用于跟踪和取得响应的结果对象。
5. 它们是分别运行的程序，本例服务端持续 `spin`，客户端处理一次响应就退出。
6. 不会。日志和服务响应是不同的事。

## 7.16 完成本章的标准

- [ ] 能运行自己编写的服务端。
- [ ] 能用命令行调用它，收到正确响应。
- [ ] 能运行自己编写的客户端，收到正确结果。
- [ ] 修改请求并重新构建后，结果随之改变。
- [ ] 能解释请求、响应、服务名称和类型。
- [ ] 能指出发送请求和计算响应分别位于哪句代码。

目前不要求独立默写整份程序。优先掌握数据怎样走、哪部分代码负责什么。

## 参考资料与验证说明

- [ROS 2 Jazzy 官方教程：C++ 服务端与客户端](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Cpp-Service-And-Client.html)
- [同一教程的 ROS 文档预览站](https://repo.test.ros2.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Cpp-Service-And-Client.html)

本笔记沿用官方教程的求和接口，教学代码使用独立命名、固定请求值和超时处理，便于逐步学习。已核对接口结构和 API 用法；本次未在你的 Ubuntu 虚拟机中编译运行，实际验证将在实操时完成。
