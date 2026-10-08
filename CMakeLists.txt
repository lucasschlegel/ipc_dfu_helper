cmake_minimum_required(VERSION 3.20.0)

list(APPEND DTC_OVERLAY_FILE "${CMAKE_SOURCE_DIR}/uart.overlay")

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(ipc_dfu_helper VERSION 1.0.0)

target_sources(app PRIVATE src/main.c)