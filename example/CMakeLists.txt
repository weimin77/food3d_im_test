cmake_minimum_required(VERSION 3.28)
project(example CXX)

find_package(food3d_im REQUIRED)

add_executable(example main.cpp)
target_compile_features(example PRIVATE cxx_std_17)
target_link_libraries(example PRIVATE food3d_im::food3d_im)

add_executable(bench bench.cpp)
target_compile_features(bench PRIVATE cxx_std_17)
target_link_libraries(bench PRIVATE food3d_im::food3d_im)
