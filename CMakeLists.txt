cmake_minimum_required(VERSION 3.9) # CMP0069 NEW

project(garbage
  DESCRIPTION "garbage demo"
  LANGUAGES CXX
)


add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/textfile.cpp
    COMMAND ${CMAKE_COMMAND} -E env echo "allo" && pwd && ${CMAKE_CURRENT_SOURCE_DIR}/script.sh && echo "file outputted"
)


add_library(joe1 STATIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/textfile.cpp> )
add_library(joe2 STATIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/textfile.cpp> )
add_library(joe3 STATIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/textfile.cpp> )
target_link_libraries(joe2 INTERFACE joe1)
target_link_libraries(joe3 INTERFACE joe2)

add_custom_target(fake DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/textfile.cpp)
add_dependencies(joe1 fake)
