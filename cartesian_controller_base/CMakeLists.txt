cmake_minimum_required(VERSION 3.10)
project(cartesian_controller_base)

set(CMAKE_CXX_STANDARD 17)

set(ADDITIONAL_COMPILE_OPTIONS -Wall -Wextra -Wpedantic -Wno-unused-parameter)
add_compile_options(${ADDITIONAL_COMPILE_OPTIONS})

# Set build type to "Release" by default
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif(NOT CMAKE_BUILD_TYPE)

# Use CMake to pass the current ROS_DISTRO via variables into a preprocessor template.
# We then include this file and switch between the different APIs.
if("$ENV{ROS_DISTRO}" STREQUAL "jazzy")
  set(CARTESIAN_CONTROLLERS_JAZZY TRUE)
elseif("$ENV{ROS_DISTRO}" STREQUAL "lyrical")
  set(CARTESIAN_CONTROLLERS_JAZZY TRUE)
elseif("$ENV{ROS_DISTRO}" STREQUAL "iron")
  set(CARTESIAN_CONTROLLERS_IRON TRUE)
elseif("$ENV{ROS_DISTRO}" STREQUAL "humble")
  set(CARTESIAN_CONTROLLERS_HUMBLE TRUE)
else()
  message(WARNING "ROS2 version must be {lyrical|jazzy|iron|humble}")
endif()
configure_file(include/cartesian_controller_base/ROS2VersionConfig.h.in ROS2VersionConfig.h)

## Find catkin macros and libraries
## if COMPONENTS list like find_package(catkin REQUIRED COMPONENTS xyz)
## is used, also find other catkin packages
find_package(Eigen3 3.3 REQUIRED NO_MODULE)
add_definitions(-DEIGEN_MPL2_ONLY)

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(controller_interface REQUIRED)
find_package(kdl_parser REQUIRED)
find_package(trajectory_msgs REQUIRED)
find_package(pluginlib REQUIRED)
find_package(urdf REQUIRED)
find_package(realtime_tools REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(hardware_interface REQUIRED)
find_package(rclcpp_lifecycle REQUIRED)


# Convenience variable for dependencies
set(THIS_PACKAGE_INCLUDE_DEPENDS
        rclcpp
        controller_interface
        kdl_parser
        trajectory_msgs
        pluginlib
        urdf
        Eigen3
        realtime_tools
        geometry_msgs
        hardware_interface
        rclcpp_lifecycle
)

#--------------------------------------------------------------------------------
# Libraries
#--------------------------------------------------------------------------------
add_library(${PROJECT_NAME} SHARED
  src/cartesian_controller_base.cpp
  src/SpatialPDController.cpp
  src/PDController.cpp
  src/IKSolver.cpp
)

# Manual includes for local directories and non-ament packages
target_include_directories(${PROJECT_NAME}
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
    ${EIGEN3_INCLUDE_DIR}
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>  # ROS2VersionConfig.h
)

target_link_libraries(${PROJECT_NAME} PUBLIC
  ${controller_interface_TARGETS}
  ${kdl_parser_TARGETS}
  ${trajectory_msgs_TARGETS}
  ${pluginlib_TARGETS}
  ${urdf_TARGETS}
  Eigen3::Eigen
  ${realtime_tools_TARGETS}
  ${rclcpp_TARGETS}
  ${geometry_msgs_TARGETS}
  ${hardware_interface_TARGETS}
  ${rclcpp_lifecycle_TARGETS}
)


add_library(ik_solvers SHARED
  src/IKSolver.cpp
  src/ForwardDynamicsSolver.cpp
  src/JacobianTransposeSolver.cpp
  src/DampedLeastSquaresSolver.cpp
  src/SelectivelyDampedLeastSquaresSolver.cpp
)

target_include_directories(ik_solvers
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>  # ROS2VersionConfig.h
)

# Prevent pluginlib from using boost
target_compile_definitions(${PROJECT_NAME} PUBLIC "PLUGINLIB__DISABLE_BOOST_FUNCTIONS")

target_link_libraries(ik_solvers PUBLIC
  ${controller_interface_TARGETS}
  ${kdl_parser_TARGETS}
  ${trajectory_msgs_TARGETS}
  ${pluginlib_TARGETS}
  ${urdf_TARGETS}
  Eigen3::Eigen
  ${realtime_tools_TARGETS}
  ${rclcpp_TARGETS}
  ${geometry_msgs_TARGETS}
  ${hardware_interface_TARGETS}
  ${rclcpp_lifecycle_TARGETS}
)


#--------------------------------------------------------------------------------
# Install and export
#--------------------------------------------------------------------------------

pluginlib_export_plugin_description_file(cartesian_controller_base ik_solver_plugin.xml)

install(
  DIRECTORY include/
  DESTINATION include
)

install(
  TARGETS ${PROJECT_NAME} ik_solvers
  EXPORT export_${PROJECT_NAME}
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
  RUNTIME DESTINATION bin
  #INCLUDES DESTINATION include
)

install(
  FILES ${CMAKE_BINARY_DIR}/ROS2VersionConfig.h
  DESTINATION include/${PROJECT_NAME}
)

ament_export_dependencies(${THIS_PACKAGE_INCLUDE_DEPENDS})
ament_export_include_directories(
  include
)
ament_export_targets(export_${PROJECT_NAME} HAS_LIBRARY_TARGET)

ament_package()
