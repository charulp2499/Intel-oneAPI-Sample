set(TARGET_NAME qrd)
set(SOURCE_FILE qrd_demo.cpp)
set(EMULATOR_TARGET ${TARGET_NAME}.fpga_emu)
set(SIMULATOR_TARGET ${TARGET_NAME}.fpga_sim)
set(FPGA_TARGET ${TARGET_NAME}.fpga)
set(FPGA_EARLY_IMAGE ${TARGET_NAME}_report.a)

# FPGA board selection
if(NOT DEFINED FPGA_DEVICE)
    set(FPGA_DEVICE "intel_a10gx_pac:pac_a10")
    message(STATUS "FPGA_DEVICE was not specified.\
                    \nConfiguring the design to run on the default FPGA board ${FPGA_DEVICE} (Intel(R) PAC with Intel Arria(R) 10 GX FPGA). \
                    \nPlease refer to the README for information on board selection.")
else()
    message(STATUS "Configuring the design to run on FPGA board ${FPGA_DEVICE}")
endif()

# This is a Windows-specific flag that enables error handling in host code
if(WIN32)
    set(AC_TYPES_COMPILE_FLAG "/Qactypes")
    set(AC_TYPES_LINK_FLAG "/Qactypes")
    set(PLATFORM_SPECIFIC_COMPILE_FLAGS "/EHsc /Wall /fp:precise -Xsfp-relaxed")
    set(PLATFORM_SPECIFIC_LINK_FLAGS "/fp:precise -Xsfp-relaxed")
    set(STACK_FLAG "/F 12582912")
else()
    set(AC_TYPES_COMPILE_FLAG "-qactypes")
    set(AC_TYPES_LINK_FLAG "")
    set(PLATFORM_SPECIFIC_COMPILE_FLAGS "-Wall -fno-finite-math-only")
    set(PLATFORM_SPECIFIC_LINK_FLAGS "")
    set(STACK_FLAG "")
endif()


# A10 parameters
set(ROWS_COMPONENT 128)
set(COLS_COMPONENT 128)
set(COMPLEX 1)
set(FIXED_ITERATIONS 64)
set(CLOCK_TARGET 360MHz)
set(SEED "-Xsseed=7")
# Overwrite design parameters according to the selected board
if(FPGA_DEVICE MATCHES ".*a10.*")
    # A10 parameters
    # Nothing to do
elseif(FPGA_DEVICE MATCHES ".*s10.*")
    # S10 parameters
    set(ROWS_COMPONENT 256)
    set(COLS_COMPONENT 256)
    set(COMPLEX 1)
    set(FIXED_ITERATIONS 110)
    set(CLOCK_TARGET 480MHz)
    set(SEED "-Xsseed=9")
elseif(FPGA_DEVICE MATCHES ".*agilex.*")
    # Agilex™ parameters
    set(ROWS_COMPONENT 256)
    set(COLS_COMPONENT 256)
    set(FIXED_ITERATIONS 110)
    set(COMPLEX 1)
    set(CLOCK_TARGET 600MHz)
    set(SEED "-Xsseed=5")
else()
    message(STATUS "Unknown board ${FPGA_DEVICE}!")
    message(STATUS "Using Arria 10 defaults.")
endif()

if(IGNORE_DEFAULT_SEED)
    set(SEED "")
endif()

if(DEFINED SET_ROWS_COMPONENT)
    set(ROWS_COMPONENT ${SET_ROWS_COMPONENT})
endif()

if(DEFINED SET_COLS_COMPONENT)
    set(COLS_COMPONENT ${SET_COLS_COMPONENT})
endif()

if(DEFINED SET_COMPLEX)
    set(COMPLEX ${SET_COMPLEX})
endif()

if(DEFINED SET_FIXED_ITERATIONS)
    set(FIXED_ITERATIONS ${SET_FIXED_ITERATIONS})
endif()

message(STATUS "ROWS_COMPONENT=${ROWS_COMPONENT}")
message(STATUS "COLS_COMPONENT=${COLS_COMPONENT}")
message(STATUS "COMPLEX=${COMPLEX}")
message(STATUS "FIXED_ITERATIONS=${FIXED_ITERATIONS}")
message(STATUS "SEED=${SEED}")

# A SYCL ahead-of-time (AoT) compile processes the device code in two stages.
# 1. The "compile" stage compiles the device code to an intermediate representation (SPIR-V).
# 2. The "link" stage invokes the compiler's FPGA backend before linking.
#    For this reason, FPGA backend flags must be passed as link flags in CMake.
set(EMULATOR_COMPILE_FLAGS "-Wall ${PLATFORM_SPECIFIC_COMPILE_FLAGS} ${AC_TYPES_COMPILE_FLAG} -Wformat-security -Werror=format-security -fbracket-depth=512 -fsycl -fintelfpga -DFIXED_ITERATIONS=${FIXED_ITERATIONS} -DCOMPLEX=${COMPLEX} -DROWS_COMPONENT=${ROWS_COMPONENT} -DCOLS_COMPONENT=${COLS_COMPONENT} -DFPGA_EMULATOR")
set(EMULATOR_LINK_FLAGS "-fsycl -fintelfpga ${PLATFORM_SPECIFIC_LINK_FLAGS} ${STACK_FLAG} ${AC_TYPES_LINK_FLAG}")
set(SIMULATOR_COMPILE_FLAGS "-fsycl -fintelfpga -Wall ${PLATFORM_SPECIFIC_COMPILE_FLAGS} ${AC_TYPES_COMPILE_FLAG} -DFPGA_SIMULATOR -fbracket-depth=512 -DFIXED_ITERATIONS=${FIXED_ITERATIONS} -DCOMPLEX=${COMPLEX} -DROWS_COMPONENT=${ROWS_COMPONENT} -DCOLS_COMPONENT=${COLS_COMPONENT} ${USER_HARDWARE_FLAGS}")
set(SIMULATOR_LINK_FLAGS "-fsycl -fintelfpga ${PLATFORM_SPECIFIC_LINK_FLAGS} ${STACK_FLAG} -Xssimulation -Xsghdl -Xsclock=${CLOCK_TARGET} -Xstarget=${FPGA_DEVICE} ${USER_SIMULATOR_FLAGS} ${AC_TYPES_LINK_FLAG}")
set(HARDWARE_COMPILE_FLAGS "-Wall ${PLATFORM_SPECIFIC_COMPILE_FLAGS} ${AC_TYPES_COMPILE_FLAG} -Wformat-security -Werror=format-security -fsycl -fintelfpga -fbracket-depth=512 -DFIXED_ITERATIONS=${FIXED_ITERATIONS} -DCOMPLEX=${COMPLEX} -DROWS_COMPONENT=${ROWS_COMPONENT} -DCOLS_COMPONENT=${COLS_COMPONENT}")
set(REPORT_LINK_FLAGS "-fsycl -fintelfpga -Xshardware ${PLATFORM_SPECIFIC_LINK_FLAGS} -Xsclock=${CLOCK_TARGET} -Xsparallel=2 ${SEED} -Xstarget=${FPGA_DEVICE} ${USER_HARDWARE_FLAGS} ${AC_TYPES_LINK_FLAG}")
set(HARDWARE_LINK_FLAGS "${REPORT_LINK_FLAGS} ${STACK_FLAG}")
# use cmake -D USER_HARDWARE_FLAGS=<flags> to set extra flags for FPGA backend compilation

###############################################################################
### FPGA Emulator
###############################################################################
add_executable(${EMULATOR_TARGET} ${SOURCE_FILE})
target_include_directories(${EMULATOR_TARGET} PRIVATE ../../../include)
set_target_properties(${EMULATOR_TARGET} PROPERTIES COMPILE_FLAGS "${EMULATOR_COMPILE_FLAGS}")
set_target_properties(${EMULATOR_TARGET} PROPERTIES LINK_FLAGS "${EMULATOR_LINK_FLAGS}")
add_custom_target(fpga_emu DEPENDS ${EMULATOR_TARGET})

###############################################################################
### Generate Report
###############################################################################
# The compile output is not an executable, but an intermediate compilation result unique to SYCL.
add_executable(${FPGA_EARLY_IMAGE} EXCLUDE_FROM_ALL ${SOURCE_FILE})
target_include_directories(${FPGA_EARLY_IMAGE} PRIVATE ../../../include)
add_custom_target(report DEPENDS ${FPGA_EARLY_IMAGE})
set_target_properties(${FPGA_EARLY_IMAGE} PROPERTIES COMPILE_FLAGS "${HARDWARE_COMPILE_FLAGS}")
set_target_properties(${FPGA_EARLY_IMAGE} PROPERTIES LINK_FLAGS "${REPORT_LINK_FLAGS} -fsycl-link=early")
# fsycl-link=early stops the compiler after RTL generation, before invoking Quartus

###############################################################################
### FPGA Simulator
###############################################################################
add_executable(${SIMULATOR_TARGET} EXCLUDE_FROM_ALL ${SOURCE_FILE})
target_include_directories(${SIMULATOR_TARGET} PRIVATE ../../../include)
add_custom_target(fpga_sim DEPENDS ${SIMULATOR_TARGET})
set_target_properties(${SIMULATOR_TARGET} PROPERTIES COMPILE_FLAGS "${SIMULATOR_COMPILE_FLAGS}")
set_target_properties(${SIMULATOR_TARGET} PROPERTIES LINK_FLAGS "${SIMULATOR_LINK_FLAGS} -reuse-exe=${CMAKE_BINARY_DIR}/${SIMULATOR_TARGET}")
# The -reuse-exe flag enables rapid recompilation of host-only code changes.
# See C++SYCL_FPGA/GettingStarted/fast_recompile for details.

###############################################################################
### FPGA Hardware
###############################################################################
add_executable(${FPGA_TARGET} EXCLUDE_FROM_ALL ${SOURCE_FILE})
target_include_directories(${FPGA_TARGET} PRIVATE ../../../include)
add_custom_target(fpga DEPENDS ${FPGA_TARGET})
set_target_properties(${FPGA_TARGET} PROPERTIES COMPILE_FLAGS "${HARDWARE_COMPILE_FLAGS}")
set_target_properties(${FPGA_TARGET} PROPERTIES LINK_FLAGS "${HARDWARE_LINK_FLAGS} -reuse-exe=${CMAKE_BINARY_DIR}/${FPGA_TARGET}")
# The -reuse-exe flag enables rapid recompilation of host-only code changes.
# See C++SYCL_FPGA/GettingStarted/fast_recompile for details.
