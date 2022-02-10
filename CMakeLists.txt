cmake_minimum_required(VERSION 3.15.0)
# cmake_policy(SET CMP0048 NEW)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
# Do not enable compiler specific extensions, for eg on GCC use -std=c++1z (=c++17) and not -std=gnu++17
set(CMAKE_CXX_EXTENSIONS OFF)

# Use ccache is available, has to be before "project()"
find_program(CCACHE_PROGRAM ccache)
if(CCACHE_PROGRAM)
  # Support Unix Makefiles and Ninja
  set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${CCACHE_PROGRAM}")
endif()

project(LinkPython VERSION 0.0.1)

message("PROJECT_BINARY_DIR=${PROJECT_BINARY_DIR}")

# Set a default build type if none was specified
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to 'Release' as none was specified.")
  set(CMAKE_BUILD_TYPE
      Release
      CACHE STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release"
                                               "MinSizeRel" "RelWithDebInfo")
endif()

#if(POLICY CMP0020)
#cmake_policy(SET CMP0020 NEW)
#endif()
#if(POLICY CMP0022)
#cmake_policy(SET CMP0022 NEW)
#endif()
#if(POLICY CMP0042)
#cmake_policy(SET CMP0042 NEW)
#endif()
#if(POLICY CMP0054)
#cmake_policy(SET CMP0054 NEW)
#endif()
#if(POLICY CMP0077)
#cmake_policy(SET CMP0077 NEW)
#endif()

add_library(project_options INTERFACE)
target_compile_features(project_options INTERFACE cxx_std_17)

###############################################################################
#                                  N I N J A                                  #
###############################################################################
# Ninja support: has to be atop for it to take effect before anything else is done
# Add Color Output if Using Ninja
macro(AddCXXFlagIfSupported flag test)
  check_cxx_compiler_flag(${flag} ${test})
  if(${${test}})
    message("adding ${flag}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${flag}")
  endif()
endmacro()

if("Ninja" STREQUAL ${CMAKE_GENERATOR})
  # Clang
  if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
    addcxxflagifsupported(-fcolor-diagnostics
                          COMPILER_SUPPORTS_fcolor-diagnostics)
  endif()

  # g++
  if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    # For some reason it doesn't say its supported, but it works...
    # AddCXXFlagIfSupported(-fdiagnostics-color COMPILER_SUPPORTS_fdiagnostics-color)
    message(STATUS "Ninja: Forcing -fdiagnostics-color=always")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fdiagnostics-color=always")
  endif()
endif()

# Xcode/Ninja generators undefined MAKE
if(CMAKE_GENERATOR MATCHES "Make")
  set(MAKE "$(MAKE)")
else()
  set(MAKE make)
endif()

###############################################################################
#                                 O P T I O N S                               #
###############################################################################

option(LINK_WITH_PYTHON "Link with Python Library for Python Plugin Builds" OFF)

###############################################################################
#          C O M P I L E R    A N D    S Y S T E M    O P T I O N S           #
###############################################################################

# Compiler and system specific options
if(UNIX)

  # all warnings
  target_compile_options(project_options INTERFACE -fPIC -fno-strict-aliasing
                                                   -Winvalid-pch)
  # Treat all warnings as errors, extra errors, and be pedantic
  target_compile_options(
    project_options INTERFACE -Wall -Wextra -Werror -pedantic-errors -pedantic
  )# Turn on warnings
  if(APPLE)
    target_compile_options(project_options INTERFACE -Wno-overloaded-virtual
                                                     -ftemplate-depth=1024)
  endif()

  # Note: CMAKE_CXX_STANDARD set to 20 should already take care of adding -std=c++20 or equivalent
endif()

###############################################################################
#                         C M A K E    C O N T R O L                          #
###############################################################################

include_directories(${CMAKE_BINARY_DIR})

# High level project configuration
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/Products")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/Products")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/Products")

set(LIBRARY_SEARCH_DIRECTORY
    "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}"
    "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/Release"
    "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/Debug")

###############################################################################
#                              E X E C U T A B L E                            #
###############################################################################

message("CMAKE_RUNTIME_OUTPUT_DIRECTORY=${CMAKE_RUNTIME_OUTPUT_DIRECTORY}")
add_executable(MyProgram main.cpp)

if(LINK_WITH_PYTHON)
  find_package(Python 3.9.9 EXACT COMPONENTS Interpreter Development REQUIRED)
  target_compile_definitions(MyProgram PRIVATE LINK_WITH_PYTHON)
  target_link_libraries(MyProgram PRIVATE project_options Python::Python)
endif()
