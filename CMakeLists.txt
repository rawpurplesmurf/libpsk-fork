cmake_minimum_required(VERSION 3.16)
project(psk C)
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)

option(PSK_BUILD_APPS "Build standalone psk_tcpmodem application" OFF)
option(PSK_BUILD_TOOLS "Build standalone CLI tools" OFF)

# ── Library ──────────────────────────────────────────────────────────────────
add_library(psk STATIC
    src/varicode.c
    src/fir.c
    src/costas.c
    src/timing.c
    src/viterbi.c
    src/tx.c
    src/rx.c
)

target_include_directories(psk
    PUBLIC  include
    PRIVATE src
)

target_link_libraries(psk PUBLIC m)

target_compile_options(psk PRIVATE
    -Wall -Wextra
    $<$<CONFIG:Release>:-O2>
)

# ── Tests ─────────────────────────────────────────────────────────────────────
enable_testing()

foreach(t varicode tx loopback snr modes)
    if(EXISTS "${CMAKE_SOURCE_DIR}/c_tests/test_${t}.c")
        add_executable(test_${t} c_tests/test_${t}.c)
        target_link_libraries(test_${t} PRIVATE psk)
        add_test(NAME ${t} COMMAND test_${t})
    endif()
endforeach()

# ── Optional Standalone Apps / Tools (Disabled by Default) ────────────────────
if(PSK_BUILD_APPS AND EXISTS "${CMAKE_SOURCE_DIR}/apps/linux_tcpmodem/main.c")
    add_executable(psk_tcpmodem apps/linux_tcpmodem/main.c)
    target_link_libraries(psk_tcpmodem PRIVATE psk)
    target_include_directories(psk_tcpmodem PRIVATE src)
endif()

if(PSK_BUILD_TOOLS AND EXISTS "${CMAKE_SOURCE_DIR}/tools/psk31_beacon.c")
    add_executable(psk31_beacon tools/psk31_beacon.c)
    target_link_libraries(psk31_beacon PRIVATE psk)
endif()

if(PSK_BUILD_TOOLS AND EXISTS "${CMAKE_SOURCE_DIR}/tools/psk31_decode.c")
    add_executable(psk31_decode tools/psk31_decode.c)
    target_link_libraries(psk31_decode PRIVATE psk)
endif()
