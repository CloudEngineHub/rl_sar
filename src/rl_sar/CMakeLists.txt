cmake_minimum_required(VERSION 3.0.2)
project(rl_sar)

add_definitions(-DCMAKE_CURRENT_SOURCE_DIR="${CMAKE_CURRENT_SOURCE_DIR}")

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(CMAKE_BUILD_TYPE Debug)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g")

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${TORCH_CXX_FLAGS}")
find_package(Torch REQUIRED)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${GAZEBO_CXX_FLAGS}")
find_package(gazebo REQUIRED)

find_package(TBB REQUIRED)

find_package(catkin REQUIRED COMPONENTS
  controller_manager
  genmsg
  joint_state_controller
  robot_state_publisher
  roscpp
  gazebo_ros
  std_msgs
  tf
  geometry_msgs
  robot_msgs
  robot_joint_controller
  rospy
)

find_package(Python3 COMPONENTS Interpreter Development REQUIRED)

catkin_package(
  CATKIN_DEPENDS
  robot_joint_controller
  rospy
)

link_directories(/usr/local/lib)
include_directories(${YAML_CPP_INCLUDE_DIR})

# Unitree A1
include_directories(library/thirdparty/unitree_legged_sdk_3.2/include)
link_directories(library/thirdparty/unitree_legged_sdk_3.2/lib)
set(UNITREE_A1_LIBS -pthread unitree_legged_sdk_amd64 lcm)

# Unitree Go2
include_directories(library/thirdparty/unitree_sdk2/include)
link_directories(library/thirdparty/unitree_sdk2/lib/x86_64)
include_directories(library/thirdparty/unitree_sdk2/thirdparty/include)
include_directories(library/thirdparty/unitree_sdk2/thirdparty/include/ddscxx)
link_directories(library/thirdparty/unitree_sdk2/thirdparty/lib/x86_64)
set(UNITREE_GO2_LIBS -pthread unitree_sdk2 ddsc ddscxx)

include_directories(
  include
  ${catkin_INCLUDE_DIRS}
  library/core/matplotlibcpp
  library/core/observation_buffer
  library/core/rl_sdk
  library/core/loop
)

add_library(rl_sdk library/core/rl_sdk/rl_sdk.cpp)
target_link_libraries(rl_sdk "${TORCH_LIBRARIES}" Python3::Python Python3::Module TBB::tbb)
set_property(TARGET rl_sdk PROPERTY CXX_STANDARD 14)
find_package(Python3 COMPONENTS NumPy)
if(Python3_NumPy_FOUND)
  target_link_libraries(rl_sdk Python3::NumPy)
else()
  target_compile_definitions(rl_sdk WITHOUT_NUMPY)
endif()

add_library(observation_buffer library/core/observation_buffer/observation_buffer.cpp)
target_link_libraries(observation_buffer "${TORCH_LIBRARIES}")
set_property(TARGET observation_buffer PROPERTY CXX_STANDARD 14)

add_executable(rl_sim src/rl_sim.cpp )
target_link_libraries(rl_sim
  ${catkin_LIBRARIES} -pthread
  rl_sdk observation_buffer yaml-cpp
)

add_executable(rl_real_a1 src/rl_real_a1.cpp )
target_link_libraries(rl_real_a1
  ${catkin_LIBRARIES} ${UNITREE_A1_LIBS}
  rl_sdk observation_buffer yaml-cpp
)

add_executable(rl_real_go2 src/rl_real_go2.cpp )
target_link_libraries(rl_real_go2
  ${catkin_LIBRARIES} ${UNITREE_GO2_LIBS}
  rl_sdk observation_buffer yaml-cpp
)

include_directories(library/thirdparty/l4w4_sdk)
add_executable(rl_real_l4w4 src/rl_real_l4w4.cpp )
target_link_libraries(rl_real_l4w4
  ${catkin_LIBRARIES} -pthread
  rl_sdk observation_buffer yaml-cpp
)

catkin_install_python(PROGRAMS
  scripts/rl_sim.py
  scripts/actuator_net.py
  DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)