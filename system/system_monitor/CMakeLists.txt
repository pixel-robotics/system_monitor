cmake_minimum_required(VERSION 3.14)
project(system_monitor)

set (CMAKE_CXX_STANDARD 17)

find_package(ament_cmake_auto REQUIRED)
find_package(diagnostic_updater REQUIRED)
find_package(system_monitor_interfaces REQUIRED)
find_package(rclcpp_components REQUIRED)


set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
find_package(NVML)
find_package(fmt REQUIRED)
set(LIBRARIES fmt)

## Specify additional locations of header files
find_path(LIBNL3_INCLUDE_DIRS
  NAMES netlink/netlink.h
  PATH_SUFFIXES libnl3
)

if(NVML_FOUND)
  include_directories(
    include
    ${LIBNL3_INCLUDE_DIRS}
    ${NVML_INCLUDE_DIRS}
  )
else()
  include_directories(
    include
    ${LIBNL3_INCLUDE_DIRS}
  )
endif()

## Declare a C++ executable

if(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
  set(CMAKE_CPU_PLATFORM "intel")
  add_definitions(-D_CPU_INTEL_)
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "arm")
  if(CMAKE_HOST_SYSTEM_VERSION MATCHES ".*raspi.*")
    set(CMAKE_CPU_PLATFORM "raspi")
    add_definitions(-D_CPU_RASPI_)
  elseif(CMAKE_HOST_SYSTEM_VERSION MATCHES ".*tegra.*")
    set(CMAKE_CPU_PLATFORM "tegra")
    add_definitions(-D_CPU_TEGRA_)
  else()
    set(CMAKE_CPU_PLATFORM "arm")
    add_definitions(-D_CPU_ARM_)
  endif()
else()
  set(CMAKE_CPU_PLATFORM "unknown")
endif()

if(NVML_FOUND)
  set(CMAKE_GPU_PLATFORM "nvml")
  add_definitions(-D_GPU_NVML_)
  set(GPU_LIBRARY ${NVML_LIBRARIES})
else()
  if(CMAKE_CPU_PLATFORM STREQUAL "tegra")
    set(CMAKE_GPU_PLATFORM "tegra")
    add_definitions(-D_GPU_TEGRA_)
  else()
    set(CMAKE_GPU_PLATFORM "unknown")
  endif()
endif()

message(STATUS "HOST_SYSTEM_VERSION: " ${CMAKE_HOST_SYSTEM_VERSION})
message(STATUS "SYSTEM_PROCESSOR: " ${CMAKE_SYSTEM_PROCESSOR})
message(STATUS "CPU PLATFORM: " ${CMAKE_CPU_PLATFORM})
message(STATUS "GPU PLATFORM: " ${CMAKE_GPU_PLATFORM})

set(CPU_MONITOR_SOURCE
  src/cpu_monitor/cpu_monitor_base.cpp
  src/cpu_monitor/${CMAKE_CPU_PLATFORM}_cpu_monitor.cpp
)

ament_auto_add_library(cpu_monitor_lib SHARED
  ${CPU_MONITOR_SOURCE}
)

ament_auto_add_library(hdd_monitor_lib SHARED
  src/hdd_monitor/hdd_monitor.cpp
)

ament_auto_add_library(mem_monitor_lib SHARED
  src/mem_monitor/mem_monitor.cpp
)

ament_auto_add_library(net_monitor_lib SHARED
  src/net_monitor/net_monitor.cpp
  src/net_monitor/nl80211.cpp
)

ament_auto_add_library(ntp_monitor_lib SHARED
  src/ntp_monitor/ntp_monitor.cpp
)

#add_library(process_monitor_lib SHARED
#  src/process_monitor/process_monitor.cpp
#)

set(GPU_MONITOR_SOURCE
  src/gpu_monitor/gpu_monitor_base.cpp
  src/gpu_monitor/${CMAKE_GPU_PLATFORM}_gpu_monitor.cpp
)
ament_auto_add_library(gpu_monitor_lib SHARED
  ${GPU_MONITOR_SOURCE}
)

ament_auto_add_library(voltage_monitor_lib SHARED
  src/voltage_monitor/voltage_monitor.cpp
)

ament_auto_add_executable(msr_reader
  reader/msr_reader/msr_reader.cpp
)

ament_auto_add_executable(hdd_reader
  reader/hdd_reader/hdd_reader.cpp
)
ament_auto_add_executable(traffic_reader
  reader/traffic_reader/traffic_reader.cpp
)

set(dependencies
  ament_index_cpp
  diagnostic_updater
  rclcpp_components
  system_monitor_interfaces
)


find_library(NL3 nl-3 REQUIRED)
find_library(NLGENL3 nl-genl-3 REQUIRED)
list(APPEND NL_LIBS ${NL3} ${NLGENL3})

ament_target_dependencies(ntp_monitor_lib
  ${dependencies}
)
ament_target_dependencies(cpu_monitor_lib
  ${dependencies}
)
ament_target_dependencies(hdd_monitor_lib
  ${dependencies}
)
ament_target_dependencies(mem_monitor_lib
  ${dependencies}
)
ament_target_dependencies(net_monitor_lib
  ${dependencies}
)
ament_target_dependencies(voltage_monitor_lib
  ${dependencies}
)
ament_target_dependencies(gpu_monitor_lib
  ${dependencies}
)

ament_target_dependencies(traffic_reader
  ${dependencies}
)



find_package(Boost REQUIRED COMPONENTS
  serialization
  thread
  filesystem
  regex
)

## Specify libraries to link a library or executable target against
target_link_libraries(voltage_monitor_lib ${Boost_LIBRARIES} ${LIBRARIES})
target_link_libraries(cpu_monitor_lib ${Boost_LIBRARIES} ${LIBRARIES})
target_link_libraries(hdd_monitor_lib ${Boost_LIBRARIES} ${LIBRARIES})
target_link_libraries(mem_monitor_lib ${LIBRARIES})
target_link_libraries(net_monitor_lib ${NL_LIBS} ${LIBRARIES})
target_link_libraries(ntp_monitor_lib ${Boost_LIBRARIES} ${LIBRARIES})
# target_link_libraries(process_monitor_lib ${LIBRARIES})
target_link_libraries(gpu_monitor_lib ${GPU_LIBRARY} ${Boost_LIBRARIES} ${LIBRARIES})
target_link_libraries(msr_reader ${Boost_LIBRARIES} ${LIBRARIES})
target_link_libraries(hdd_reader ${Boost_LIBRARIES} ${LIBRARIES})
target_link_libraries(traffic_reader ${Boost_LIBRARIES} ${LIBRARIES})


rclcpp_components_register_node(cpu_monitor_lib
  PLUGIN "CPUMonitor"
  EXECUTABLE cpu_monitor
)

rclcpp_components_register_node(hdd_monitor_lib
  PLUGIN "HddMonitor"
  EXECUTABLE hdd_monitor
)

rclcpp_components_register_node(mem_monitor_lib
  PLUGIN "MemMonitor"
  EXECUTABLE mem_monitor
)
rclcpp_components_register_node(net_monitor_lib
  PLUGIN "NetMonitor"
  EXECUTABLE net_monitor
)

rclcpp_components_register_node(ntp_monitor_lib
  PLUGIN "NTPMonitor"
  EXECUTABLE ntp_monitor
)

#rclcpp_components_register_node(process_monitor_lib
#  PLUGIN "ProcessMonitor"
##  EXECUTABLE process_monitor
#)

rclcpp_components_register_node(gpu_monitor_lib
  PLUGIN "GPUMonitor"
  EXECUTABLE gpu_monitor
)

rclcpp_components_register_node(voltage_monitor_lib
  PLUGIN "VoltageMonitor"
  EXECUTABLE voltage_monitor
)

ament_auto_package(INSTALL_TO_SHARE config)
