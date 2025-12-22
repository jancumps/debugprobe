cmake_minimum_required(VERSION 3.12)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 26)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fmodules-ts -fcommon -fno-rtti -fno-exceptions")
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# path to PICO utilities (if needed)
if (NOT DEFINED ENV{PIOASM_DIR})
        message("PIOASM: set PIOASM_DIR env variable if you use PIO")
else()
        message("PIOASM: PIOASM_DIR env variable is set to '$ENV{PIOASM_DIR}'")
        set(pioasm_DIR $ENV{PIOASM_DIR})
endif()
if (NOT DEFINED ENV{PICOTOOL_DIR})
        message("PICOTOOL: set PICOTOOL_DIR env variable if you create uf2 output with pico_add_extra_outputs()")
else()
        message("PICOTOOL: PICOTOOL_DIR env variable is set to '$ENV{PICOTOOL_DIR}'")
        set(picotool_DIR $ENV{PICOTOOL_DIR})
endif()

set(PICO_PLATFORM rp2040 CACHE STRING "Pico Platform")
set(PICO_BOARD seeed_xiao_rp2040 CACHE STRING "Board type")

include(pico_sdk_import.cmake)

set(FREERTOS_KERNEL_PATH ${CMAKE_CURRENT_LIST_DIR}/freertos)
include(FreeRTOS_Kernel_import.cmake)

project(debugprobe)

pico_sdk_init()

if (${PICO_SDK_VERSION_MAJOR} LESS 2)
	message(SEND_ERROR "Version 2 of the Pico SDK is required to compile this project. Please update your installation at ${PICO_SDK_PATH}")
endif ()

add_executable(debugprobe
        src/probe_config.c
        src/main.c
        src/usb_descriptors.c
        src/probe.c
        src/cdc_uart.c
        src/get_serial.c
        src/sw_dp_pio.c
        src/tusb_edpt_handler.c
)

target_sources(debugprobe PRIVATE
        CMSIS_5/CMSIS/DAP/Firmware/Source/DAP.c
        CMSIS_5/CMSIS/DAP/Firmware/Source/JTAG_DP.c
        CMSIS_5/CMSIS/DAP/Firmware/Source/DAP_vendor.c
        CMSIS_5/CMSIS/DAP/Firmware/Source/SWO.c
        #CMSIS_5/CMSIS/DAP/Firmware/Source/SW_DP.c
        )

target_include_directories(debugprobe PRIVATE
        CMSIS_5/CMSIS/DAP/Firmware/Include/
        CMSIS_5/CMSIS/Core/Include/
        include/
        )

target_compile_options(debugprobe PRIVATE -Wall)

pico_generate_pio_header(debugprobe ${CMAKE_CURRENT_LIST_DIR}/src/probe.pio)
pico_generate_pio_header(debugprobe ${CMAKE_CURRENT_LIST_DIR}/src/probe_oen.pio)

target_include_directories(debugprobe PRIVATE src)

target_compile_definitions (debugprobe PRIVATE
	PICO_RP2040_USB_DEVICE_ENUMERATION_FIX=1
)

option (DEBUG_ON_PICO "Compile firmware for the Pico instead of Debug Probe" OFF)
if (DEBUG_ON_PICO)
    target_compile_definitions (debugprobe PRIVATE
	DEBUG_ON_PICO=1
    )
    if (PICO_BOARD STREQUAL "pico")
        set_target_properties(debugprobe PROPERTIES
            OUTPUT_NAME "debugprobe_on_pico"
        )
    elseif (PICO_BOARD STREQUAL "seeed_xiao_rp2040")
        set_target_properties(debugprobe PROPERTIES
            OUTPUT_NAME "debugprobe_on_xiao"
        )
    elseif (PICO_BOARD STREQUAL "pico2")
        set_target_properties(debugprobe PROPERTIES
            OUTPUT_NAME "debugprobe_on_pico2"
        )
    else ()
        message(SEND_ERROR "Unsupported board ${PICO_BOARD}")
    endif ()
endif ()


target_link_libraries(debugprobe PRIVATE
        pico_multicore
        pico_stdlib
        pico_unique_id
        tinyusb_device
        tinyusb_board
        hardware_pio
        FreeRTOS-Kernel
        FreeRTOS-Kernel-Heap1
)

pico_set_binary_type(debugprobe copy_to_ram)

pico_add_extra_outputs(debugprobe)
