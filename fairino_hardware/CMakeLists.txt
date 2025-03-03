cmake_minimum_required(VERSION 3.8)
project(fairino_hardware)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic -fPIC -g -ggdb3)
endif()

set(CMAKE_BUILD_TYPE Release)
set(CMAKE_CXX_STANDARD 17)

# find dependencies
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)
find_package(hardware_interface REQUIRED)
find_package(pluginlib REQUIRED)
find_package(rclcpp_lifecycle REQUIRED)
find_package(fairino_msgs REQUIRED)

# Find the Fairino library
add_library(FAIRINO_LIBRARY SHARED IMPORTED)
set_target_properties(FAIRINO_LIBRARY PROPERTIES IMPORTED_LOCATION ${CMAKE_CURRENT_SOURCE_DIR}/libfairino/lib/libfairino.so.2)

# Set variable
file(GLOB SRC_FILES ./src/command_server.cpp ./src/command_server_node.cpp)
file(GLOB EXP_SRC_FILES ./examples/src/*.cpp)

# Compile
add_executable(ros2_cmd_server ${SRC_FILES})
add_executable(srv_test_client ${EXP_SRC_FILES})
add_library(
  fairino_hardware
  SHARED
  ./src/fairino_hardware_interface.cpp
)

# Add header file source
target_include_directories(ros2_cmd_server PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include/ ${CMAKE_CURRENT_SOURCE_DIR}/)
target_include_directories(srv_test_client PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/examples/include)
target_include_directories(fairino_hardware PUBLIC
$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/>
$<INSTALL_INTERFACE:include/fairino_hardware>
)

# Link libraries correctly
message(STATUS "FAIRINO_LIBRARY found at: ${FAIRINO_LIBRARY}")

target_link_libraries(ros2_cmd_server PUBLIC FAIRINO_LIBRARY)
target_link_libraries(fairino_hardware PUBLIC FAIRINO_LIBRARY)

# Add dependencies
ament_target_dependencies(ros2_cmd_server PUBLIC rclcpp fairino_msgs)
ament_target_dependencies(srv_test_client PUBLIC rclcpp fairino_msgs)
ament_target_dependencies(fairino_hardware PUBLIC
  hardware_interface
  pluginlib
  rclcpp
  fairino_msgs
  rclcpp_lifecycle
)

# Export hardware plugins
pluginlib_export_plugin_description_file(hardware_interface fairino_hardware.xml)

# Install all files
install(TARGETS ros2_cmd_server
  DESTINATION lib/${PROJECT_NAME}
)

install(TARGETS srv_test_client
  DESTINATION lib/${PROJECT_NAME}
)

install(TARGETS fairino_hardware
  EXPORT fairino_hardware
  ARCHIVE DESTINATION lib/${PROJECT_NAME}
  LIBRARY DESTINATION lib/${PROJECT_NAME}
)

install(
  IMPORTED_RUNTIME_ARTIFACTS FAIRINO_LIBRARY
  DESTINATION lib)

# Copy Fairino SDK library file to specific path
file(MAKE_DIRECTORY "${CMAKE_INSTALL_PREFIX}/lib/${PROJECT_NAME}")

file(COPY_FILE "libfairino/lib/libfairino.so.2.1.4"
"${CMAKE_INSTALL_PREFIX}/lib/${PROJECT_NAME}/libfairino.so.2.1.4"
)

file(CREATE_LINK
"${CMAKE_INSTALL_PREFIX}/lib/${PROJECT_NAME}/libfairino.so.2.1.4"
"${CMAKE_INSTALL_PREFIX}/lib/${PROJECT_NAME}/libfairino.so.2"
SYMBOLIC)

file(CREATE_LINK
"${CMAKE_INSTALL_PREFIX}/lib/${PROJECT_NAME}/libfairino.so.2"
"${CMAKE_INSTALL_PREFIX}/lib/${PROJECT_NAME}/libfairino.so"
SYMBOLIC)

# Set LD_LIBRARY_PATH dynamically
set(FAIRINO_LIBRARY_DIR ${CMAKE_INSTALL_PREFIX}/lib/${PROJECT_NAME})
set(ENV{LD_LIBRARY_PATH} "${FAIRINO_LIBRARY_DIR}:$ENV{LD_LIBRARY_PATH}")

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  find_package(hardware_interface REQUIRED)
  set(ament_cmake_copyright_FOUND TRUE)
  set(ament_cmake_cpplint_FOUND TRUE)
  ament_lint_auto_find_test_dependencies()
endif()

# Export
ament_export_targets(fairino_hardware HAS_LIBRARY_TARGET)
ament_export_dependencies(ament_cmake rclcpp hardware_interface pluginlib rclcpp_lifecycle)

ament_package()
