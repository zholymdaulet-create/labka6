cmake_minimum_required(VERSION 3.16)
project(week6 CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

find_package(glfw3 REQUIRED)
find_package(OpenGL REQUIRED)

add_executable(week6 src/main.cpp)
target_link_libraries(week6 PRIVATE glfw OpenGL::GL)
