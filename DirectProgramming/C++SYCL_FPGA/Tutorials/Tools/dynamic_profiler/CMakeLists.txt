if(UNIX)
    # Direct CMake to use icpx rather than the default C++ compiler/linker
    set(CMAKE_CXX_COMPILER icpx)
    cmake_minimum_required (VERSION 3.4)

    project(DynamicProfiler CXX)

    set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})

    add_subdirectory (src)
else() # Windows
    # This code sample does not currently support windows
    message(FATAL_ERROR "Error: This code sample cannot be configured to run on Windows!\
                    \nPlease look over the README for information on using this feature on windows.")
endif()
