cmake_minimum_required(VERSION 3.16)

project(minicoropp LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_SCAN_FOR_MODULES OFF)

include(get_cpm.cmake)

option(MINICOROPP_USE_ASM "Use ASM implementation for coroutine context switching" ON)
option(MINICOROPP_NO_MULTITHREAD "Disable thread-local storage for minicoropp" OFF)
option(MINICOROPP_NO_DEFAULT_ALLOCATOR "Disable default mco allocator" OFF)
option(MINICOROPP_USE_VMEM_ALLOCATOR "Use Virtual Memory allocator instead of malloc" OFF)

set(MINICOROPP_MIN_STACK_SIZE 32768 CACHE STRING "Minimum stack size for coroutines")
set(MINICOROPP_DEFAULT_STACK_SIZE 65536 CACHE STRING "Default stack size for coroutines")
set(MINICOROPP_DEFAULT_STORAGE_SIZE 1024 CACHE STRING "Default storage size reserved for coroutine buffers")

# ------------------------------------------------------------------------------
# Generated configuration
# ------------------------------------------------------------------------------

set(MINICOROPP_GENERATED_INCLUDE_DIR
        ${CMAKE_CURRENT_BINARY_DIR}/include
)

configure_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/minicoro_config.h.in
        ${MINICOROPP_GENERATED_INCLUDE_DIR}/minicoropp/minicoro_config.h
)

# ------------------------------------------------------------------------------
# Headers
# ------------------------------------------------------------------------------

file(GLOB_RECURSE HEADERS CONFIGURE_DEPENDS
        ${CMAKE_CURRENT_SOURCE_DIR}/src/*.hpp
)

foreach(header ${HEADERS})
    file(RELATIVE_PATH rel_path
            ${CMAKE_CURRENT_SOURCE_DIR}/src
            ${header}
    )

    set(dest
            ${MINICOROPP_GENERATED_INCLUDE_DIR}/minicoropp/${rel_path}
    )

    get_filename_component(dest_dir ${dest} DIRECTORY)
    file(MAKE_DIRECTORY ${dest_dir})

    configure_file(
            ${header}
            ${dest}
            COPYONLY
    )
endforeach()

# ------------------------------------------------------------------------------
# Library
# ------------------------------------------------------------------------------

add_library(minicoropp STATIC
        src/coroutine.cpp
        src/coroutine.hpp
        src/mutex.cpp
        src/mutex.hpp
)

add_library(minicoropp::minicoropp ALIAS minicoropp)

target_include_directories(minicoropp
        PUBLIC
        $<BUILD_INTERFACE:${MINICOROPP_GENERATED_INCLUDE_DIR}>
        $<INSTALL_INTERFACE:include>
)

# ------------------------------------------------------------------------------
# Compiler configuration
# ------------------------------------------------------------------------------

if(MSVC)
    target_compile_options(minicoropp PRIVATE
            /wd4068
            /diagnostics:caret
            /W4
            /permissive-
            /WX
    )

    target_compile_definitions(minicoropp PRIVATE
            WIN32_LEAN_AND_MEAN
            NOMINMAX
            CRT_SECURE_NO_WARNINGS
    )
else()
    target_compile_options(minicoropp PRIVATE
            -Wall
            -Wextra
            -Wpedantic
            -Werror
    )
endif()

set_property(
        TARGET minicoropp
        PROPERTY COMPILE_WARNING_AS_ERROR ON
)

target_precompile_headers(minicoropp PRIVATE
        src/pch.hpp
)

# ------------------------------------------------------------------------------
# Dependencies
# ------------------------------------------------------------------------------

include(vendor/minicoro.cmake)

target_link_libraries(minicoropp
        PRIVATE
        minicoro::minicoro
)

# ------------------------------------------------------------------------------
# Tests
#
# Only add tests when minicoropp itself is the top-level project.
# This prevents consumers using FetchContent_MakeAvailable(minicoropp)
# from getting minicoropp's tests pulled into their build.
# ------------------------------------------------------------------------------

if(PROJECT_IS_TOP_LEVEL)
    enable_testing()
    add_subdirectory(test)
endif()
