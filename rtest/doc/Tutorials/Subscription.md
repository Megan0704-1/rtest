# Writing a Subscription test

```{contents} Contents
---
depth: 2
local: true
---
```

## Background

In this tutorial we will write a simple test suite that verifies that a ROS 2 Node implementation that is using a Subscription works as expected.

``rtest`` allows white-box access to subscriptions via ``findSubscription`` API, enabling isolated and deterministic tests without requiring ROS 2 executors or spinning threads. This means all messages are delivered immediately, with no latency, when the test code calls the `rtest` API.

In this example, we shall
- Desmonstrate a simple subscriber to ``/test_topic`` using the default ``SensorDataQoS`` profile.
- Use the ``rtest`` subscription framework to inject messages directly into the subscriber without passing through the ROS middleware.
- Verify that the callback executed as expected by inspecting the member variable ``lastMsg_``.

## Prerequisites

[Writing a simple publisher and subscriber (C++)](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Cpp-Publisher-And-Subscriber.html#)

[Writing Basic Tests with C++ with GTest](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Testing/Cpp.html)


## Tasks

### 1 Create a package

Navigate to your ROS 2 workspace sources, e.g. `ros2_ws/src`, and run the package creation command:

```shell
$ ros2 pkg create example_app --dependencies rclcpp std_msgs
```

Navigate to `example_app`.


### 2 Write the subscriber node

Add the `Subscriber` class definition in `include/example_app/subscriber.hpp` with the following code:

```c++
#pragma once

#include <rclcpp/rclcpp.hpp>
#include <std_msgs/msg/string.hpp>

class Subscriber : public rclcpp::Node
{
public:
  explicit Subscriber();

  const std_msgs::msg::String & getLastMsg() const { return lastMsg_; }

private:
  rclcpp::Subscription<std_msgs::msg::String>::SharedPtr subscription;
  std_msgs::msg::String lastMsg_{};
};
```

And add a class implementation in `src/subscriber.cpp`:

```c++
#include "example_app/subscriber.hpp"

Subscriber::Subscriber()
: rclcpp::Node("test_subscriber") {
  subscription = create_subscription<std_msgs::msg::String>("test_topic",    // (1)
      rclcpp::SensorDataQoS(),
      [this](std_msgs::msg::String::UniquePtr msg) {                         // (2)
      lastMsg_ = *msg;
    });
}
```

Open the `CMakeLists.txt` and add the Subscriber as a library:
- **Note:** `ament_target_dependencies` is [depricated](https://docs.ros.org/en/kilted/Releases/Release-Kilted-Kaiju.html#ament-target-dependencies-is-deprecated) so we use plain CMake `target_link_libraries`

```cmake
cmake_minimum_required(VERSION 3.8)
project(example_app)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# find dependencies
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)

# Add Subscriber
add_library(subscriber src/subscriber.cpp)

target_include_directories(subscriber PUBLIC
  ${CMAKE_CURRENT_SOURCE_DIR}/include
)

target_link_libraries(subscriber
  rclcpp::rclcpp
  ${std_msgs_TARGETS}
)

ament_package()
```


### 3 Examine the code

```c++
subscription = create_subscription<std_msgs::msg::String>("test_topic",    // (1)
```
The Node creates a subscription with msg type `std_msgs::msg::String` and topic name `test_topic`.

```c++
[this](std_msgs::msg::String::UniquePtr msg) {                         // (2)
      lastMsg_ = *msg;
```
The subscription callback is a lambda that stores the received message in `lastMsg_` member.


### 4 Add unit tests

#### 4.1 Add dependency to `rtest`


Open the `package.xml` and add the `rtest` test dependency:

```xml
<package format="3">
  ...
  <test_depend>rtest</test_depend>
</package>
```

**NOTE** Currently `rtest` supports writing tests with GTest/GMock only. There's no need to add that dependencies explicitly.

#### 4.2 Implement a simple unit test

Create the `test` directory and add a C++ tests implementation file `test/subscriber_test.cpp`

```c++
#include <gtest/gtest.h>
#include "example_app/subscriber.hpp"

TEST(Subscription, NodeCreatesSubscriptionAndReceivesMessage)
{
  // Create tested Node
  auto node = std::make_shared<Subscriber>();

  // Retrieve the subscription created by the Node
  auto subscription = rtest::findSubscription<std_msgs::msg::String>(node, "/test_topic");

  // Verify that the Node actually created the Subscription with topic: "/test_topic"
  ASSERT_TRUE(subscription);

  // Assert that initially the stored message is empty
  ASSERT_TRUE(node->getLastMsg().data.empty());

  // Inject a message to the subscription
  auto msg = std::make_shared<std_msgs::msg::String>();
  msg->set__data("test_msg");
  subscription->handle_message(msg);

  // check the stored message
  EXPECT_EQ(node->getLastMsg().data, "test_msg");
}

```

Create the main test runner in `test/main.cpp`:

```c++
#include <gmock/gmock.h>
#include <rclcpp/rclcpp.hpp>

int main(int argc, char ** argv)
{
  // Initialize Google Test and ROS2
  ::testing::InitGoogleMock(&argc, argv);
  rclcpp::init(argc, argv);

  // Run all the tests
  int result = RUN_ALL_TESTS();

  // Shutdown ROS2
  rclcpp::shutdown();
  return result;
}
```

#### 4.3 Add tests to CMakeLists.txt

Create the `test/CMakeLists.txt` file:

```cmake
find_package(rtest REQUIRED)
find_package(ament_cmake_gmock REQUIRED)

ament_add_gmock(${PROJECT_NAME}-test
    main.cpp
    ${CMAKE_SOURCE_DIR}/src/subscriber.cpp
    subscriber_test.cpp
)

target_include_directories(${PROJECT_NAME}-test PRIVATE
  ${CMAKE_SOURCE_DIR}/include
)

# When linking against `std_msgs`, note that this package is an exception to the standard CMake target naming convention. Unlike most packages that provide a single target like `package::package`, `std_msgs` does not currently provide a `std_msgs::std_msgs` target.
target_link_libraries(${PROJECT_NAME}-test
  rclcpp::rclcpp
  rclcpp_components::component
  ${std_msgs_TARGETS}
)
```

Update the root `CMakeLists.txt` with:

```cmake
if(BUILD_TESTING)
  add_subdirectory(test)
endif()

ament_package() # Must be the last statement
```


### 4.4 Build and run the tests

Build the `example_app` package:

```shell
$ colcon build --packages-up-to example_app --cmake-args -DBUILD_TESTING=On
```

Run the tests:

```shell
$ colcon test --packages-select example_app --event-handlers console_cohesion+
```

## Key Concepts

- `rtest::findSubscription` locates a Subscription instance for testing.
- `handle_message` is used to simulate message reception without a running ROS 2 system.
- Tests use standard GoogleTest (gtest) and GoogleMock (gmock) macros.
- **Note:** Other test frameworks (e.g., Catch2) are not currently supported.

Try It Yourself!
