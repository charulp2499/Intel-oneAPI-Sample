set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsycl")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS}")

add_executable (03_sycl_migrated src/main.cpp src/jacobi.cpp)
target_link_libraries(03_sycl_migrated sycl)

add_custom_target (run_cpu ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/03_sycl_migrated)
add_custom_target (run_gpu  SYCL_DEVICE_FILTER=gpu ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/03_sycl_migrated)

