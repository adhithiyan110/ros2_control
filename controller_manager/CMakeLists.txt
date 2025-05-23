cmake_minimum_required(VERSION 3.16)
project(controller_manager LANGUAGES CXX)

find_package(ros2_control_cmake REQUIRED)
set_compiler_options()
export_windows_symbols()

set(THIS_PACKAGE_INCLUDE_DEPENDS
  controller_interface
  controller_manager_msgs
  diagnostic_updater
  hardware_interface
  pluginlib
  rclcpp
  realtime_tools
  std_msgs
  libstatistics_collector
  generate_parameter_library
  fmt
)

find_package(ament_cmake REQUIRED)
find_package(ament_cmake_python REQUIRED)
find_package(ament_cmake_core REQUIRED)
find_package(ament_cmake_gen_version_h REQUIRED)
find_package(backward_ros REQUIRED)
foreach(Dependency IN ITEMS ${THIS_PACKAGE_INCLUDE_DEPENDS})
  find_package(${Dependency} REQUIRED)
endforeach()

generate_parameter_library(controller_manager_parameters
  src/controller_manager_parameters.yaml
)

add_library(controller_manager SHARED
  src/controller_manager.cpp
)
target_compile_features(controller_manager PUBLIC cxx_std_17)
target_include_directories(controller_manager PUBLIC
  $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include/controller_manager>
)
target_link_libraries(controller_manager PUBLIC
  controller_manager_parameters
)
ament_target_dependencies(controller_manager PUBLIC ${THIS_PACKAGE_INCLUDE_DEPENDS})

add_executable(ros2_control_node src/ros2_control_node.cpp)
target_link_libraries(ros2_control_node PRIVATE
  controller_manager
)

if(BUILD_TESTING)
  find_package(ament_cmake_gmock REQUIRED)
  find_package(ros2_control_test_assets REQUIRED)
  find_package(example_interfaces REQUIRED)

  # Plugin Libraries that are built and installed for use in testing
  add_library(test_controller SHARED test/test_controller/test_controller.cpp)
  target_link_libraries(test_controller PUBLIC controller_manager)
  ament_target_dependencies(test_controller PUBLIC example_interfaces)
  pluginlib_export_plugin_description_file(controller_interface test/test_controller/test_controller.xml)
  install(
    TARGETS test_controller
    DESTINATION lib
  )

  add_library(test_controller_failed_init SHARED
    test/test_controller_failed_init/test_controller_failed_init.cpp
  )
  target_link_libraries(test_controller_failed_init PUBLIC controller_manager)
  pluginlib_export_plugin_description_file(
    controller_interface test/test_controller_failed_init/test_controller_failed_init.xml)
  install(
    TARGETS test_controller_failed_init
    DESTINATION lib
  )

  add_library(test_chainable_controller SHARED
    test/test_chainable_controller/test_chainable_controller.cpp
  )
  ament_target_dependencies(test_chainable_controller PUBLIC realtime_tools)
  target_link_libraries(test_chainable_controller PUBLIC controller_manager)
  pluginlib_export_plugin_description_file(
    controller_interface test/test_chainable_controller/test_chainable_controller.xml)
  install(
    TARGETS test_chainable_controller
    DESTINATION lib
  )

  ament_add_gmock(test_controller_manager
    test/test_controller_manager.cpp
    TIMEOUT 180
  )
  target_link_libraries(test_controller_manager
    controller_manager
    test_controller
    test_chainable_controller
    ros2_control_test_assets::ros2_control_test_assets
  )

  ament_add_gmock(test_controller_manager_with_namespace
    test/test_controller_manager_with_namespace.cpp
  )
  target_link_libraries(test_controller_manager_with_namespace
    controller_manager
    test_controller
    ros2_control_test_assets::ros2_control_test_assets
  )

  ament_add_gmock(test_controller_manager_hardware_error_handling
    test/test_controller_manager_hardware_error_handling.cpp
  )
  target_link_libraries(test_controller_manager_hardware_error_handling
    controller_manager
    test_controller
    ros2_control_test_assets::ros2_control_test_assets
  )

  ament_add_gmock(test_load_controller
    test/test_load_controller.cpp
    APPEND_ENV AMENT_PREFIX_PATH=${ament_index_build_path}_$<CONFIG>
  )
  target_link_libraries(test_load_controller
    controller_manager
    test_controller
    test_controller_failed_init
    ros2_control_test_assets::ros2_control_test_assets
  )

  ament_add_gmock(test_controllers_chaining_with_controller_manager
    test/test_controllers_chaining_with_controller_manager.cpp
  )
  target_link_libraries(test_controllers_chaining_with_controller_manager
    controller_manager
    test_chainable_controller
    test_controller
    ros2_control_test_assets::ros2_control_test_assets
  )

  ament_add_gmock(test_controller_manager_srvs
    test/test_controller_manager_srvs.cpp
    APPEND_ENV AMENT_PREFIX_PATH=${ament_index_build_path}_$<CONFIG>
  )
  target_link_libraries(test_controller_manager_srvs
    controller_manager
    test_controller
    test_chainable_controller
    ros2_control_test_assets::ros2_control_test_assets
  )
  set_tests_properties(test_controller_manager_srvs PROPERTIES TIMEOUT 120)
  ament_target_dependencies(test_controller_manager_srvs
    controller_manager_msgs
  )
  ament_add_gmock(test_controller_manager_urdf_passing
    test/test_controller_manager_urdf_passing.cpp
  )
  target_link_libraries(test_controller_manager_urdf_passing
    controller_manager
    test_controller
    ros2_control_test_assets::ros2_control_test_assets
  )
  ament_target_dependencies(test_controller_manager_urdf_passing
    controller_manager_msgs
  )

  add_library(test_controller_with_interfaces SHARED
    test/test_controller_with_interfaces/test_controller_with_interfaces.cpp
  )
  target_link_libraries(test_controller_with_interfaces PUBLIC
    controller_manager
  )
  pluginlib_export_plugin_description_file(
    controller_interface test/test_controller_with_interfaces/test_controller_with_interfaces.xml)
  install(
    TARGETS test_controller_with_interfaces
    DESTINATION lib
  )

  ament_add_gmock(test_release_interfaces
    test/test_release_interfaces.cpp
    APPEND_ENV AMENT_PREFIX_PATH=${ament_index_build_path}_$<CONFIG>
  )
  target_link_libraries(test_release_interfaces
    controller_manager
    test_controller
    test_controller_with_interfaces
    ros2_control_test_assets::ros2_control_test_assets
  )

  ament_add_gmock(test_spawner_unspawner
    test/test_spawner_unspawner.cpp
    TIMEOUT 180
  )
  target_link_libraries(test_spawner_unspawner
    controller_manager
    test_controller
    ros2_control_test_assets::ros2_control_test_assets
  )
  target_compile_definitions(
    test_spawner_unspawner
    PRIVATE PARAMETERS_FILE_PATH="${CMAKE_CURRENT_LIST_DIR}/test/")

  ament_add_gmock(test_hardware_spawner
    test/test_hardware_spawner.cpp
    TIMEOUT 120
  )
  target_link_libraries(test_hardware_spawner
    controller_manager
    test_controller
    ros2_control_test_assets::ros2_control_test_assets
  )

  ament_add_gmock(test_hardware_management_srvs
    test/test_hardware_management_srvs.cpp
  )
  target_link_libraries(test_hardware_management_srvs
    controller_manager
    test_controller
    ros2_control_test_assets::ros2_control_test_assets
  )
  ament_target_dependencies(test_hardware_management_srvs
    controller_manager_msgs
  )

  find_package(ament_cmake_pytest REQUIRED)
  install(FILES test/test_ros2_control_node.yaml
    DESTINATION test)
  ament_add_pytest_test(test_ros2_control_node test/test_ros2_control_node_launch.py)
endif()

install(
  DIRECTORY include/
  DESTINATION include/controller_manager
)
install(
  TARGETS controller_manager controller_manager_parameters
  EXPORT export_controller_manager
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
)
install(
  TARGETS ros2_control_node
  RUNTIME DESTINATION lib/controller_manager
)

ament_python_install_package(controller_manager
  SCRIPTS_DESTINATION lib/controller_manager
)
ament_export_targets(export_controller_manager HAS_LIBRARY_TARGET)
ament_export_dependencies(${THIS_PACKAGE_INCLUDE_DEPENDS})
ament_package()
ament_generate_version_header(${PROJECT_NAME})
