cmake_minimum_required(VERSION 3.8)
project(rosaria2)

if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 14)
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# find dependencies
find_package(ament_cmake REQUIRED)
# uncomment the following section in order to fill in
# further dependencies manually.
find_package(rclcpp REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(std_msgs REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(nav_msgs REQUIRED)
find_package(std_srvs REQUIRED)
find_package(tf2 REQUIRED)
find_package(tf2_ros REQUIRED)
find_package(tf2_geometry_msgs REQUIRED)
# find_package(aria REQUIRED)  --> system package, not configured?
find_package(rosidl_default_generators REQUIRED)  # for message generation

# ARIA (MobileRobots) — must resolve at configure time and record RPATH for installed binaries
set(ARIA_LIB_HINTS "")
if(DEFINED ARIA_LIB_DIR AND ARIA_LIB_DIR)
  list(APPEND ARIA_LIB_HINTS "${ARIA_LIB_DIR}")
endif()
list(APPEND ARIA_LIB_HINTS
  /usr/local/lib
  /usr/lib
  /usr/lib/aarch64-linux-gnu
  /usr/lib/x86_64-linux-gnu)
find_library(ARIA_SHARED_LIB
  NAMES Aria libAria.so.3 libAria.so libAria
  PATHS ${ARIA_LIB_HINTS}
  NO_DEFAULT_PATH)
if(NOT ARIA_SHARED_LIB)
  find_library(ARIA_SHARED_LIB NAMES Aria libAria.so.3 libAria.so libAria)
endif()
if(NOT ARIA_SHARED_LIB)
  message(FATAL_ERROR "libAria not found. Install ARIA SDK / set LD_LIBRARY_PATH, or pass -DARIA_LIB_DIR=/path/to/dir")
endif()
get_filename_component(ARIA_LIBRARY_DIR "${ARIA_SHARED_LIB}" DIRECTORY)

# generate messages
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/BumperState.msg"
  DEPENDENCIES std_msgs # Add packages that above messages depend on, in this case geometry_msgs for Sphere.msg
)

# test binary
add_executable(${PROJECT_NAME}_debug src/test.cpp src/rosaria2_node.cpp src/laser_publisher.cpp)
target_include_directories(${PROJECT_NAME}_debug
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<BUILD_INTERFACE:/usr/local/include/Aria>
    $<INSTALL_INTERFACE:include>)

target_link_libraries(${PROJECT_NAME}_debug "${ARIA_SHARED_LIB}")
set_target_properties(${PROJECT_NAME}_debug PROPERTIES
  BUILD_RPATH "${ARIA_LIBRARY_DIR}"
  INSTALL_RPATH "${ARIA_LIBRARY_DIR}")
ament_target_dependencies(${PROJECT_NAME}_debug rclcpp geometry_msgs std_msgs sensor_msgs nav_msgs std_srvs tf2 tf2_ros tf2_geometry_msgs)

# declare generated messages as dependencies to force generation before compilation
# cf. https://discourse.ros.org/t/ros2-how-to-use-custom-message-in-project-where-its-declared/2071
rosidl_target_interfaces(${PROJECT_NAME}_debug ${PROJECT_NAME} "rosidl_typesupport_cpp")

# install target
install(TARGETS
  ${PROJECT_NAME}_debug
  DESTINATION lib/${PROJECT_NAME})


if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  # the following line skips the linter which checks for copyrights
  # comment the line when a copyright and license is added to all source files
  set(ament_cmake_copyright_FOUND TRUE)
  # the following line skips cpplint (only works in a git repo)
  # comment the line when this package is in a git repo and when
  # a copyright and license is added to all source files
  set(ament_cmake_cpplint_FOUND TRUE)
  ament_lint_auto_find_test_dependencies()
endif()

ament_package()
