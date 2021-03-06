set( CONFORMANCE_SUFFIX "" )
set(CLConform_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})

project(CLConform${CONFORMANCE_SUFFIX})

if(CMAKE_BUILD_TYPE STREQUAL "release")
    set (BUILD_FLAVOR "release")
else(CMAKE_BUILD_TYPE STREQUAL "release")
    set (BUILD_FLAVOR "debug")
endif(CMAKE_BUILD_TYPE STREQUAL "release")

set(CLConform_VERSION_MAJOR "2")
set(CLConform_VERSION_MINOR "2")
set(CLConform_VERSION_MICRO "0")
set(CLConform_VERSION_EXTRA "")
set(CLConform_VERSION "${CLConform_VERSION_MAJOR}.${CLConform_VERSION_MINOR}")
set(CLConform_VERSION_FULL
    "${CLConform_VERSION}.${CLConform_VERSION_MICRO}${CLConform_VERSION_EXTRA}")

cmake_minimum_required(VERSION 3.1)

add_definitions(-DCL_USE_DEPRECATED_OPENCL_2_1_APIS=1)
add_definitions(-DCL_USE_DEPRECATED_OPENCL_2_0_APIS=1)
add_definitions(-DCL_USE_DEPRECATED_OPENCL_1_2_APIS=1)
add_definitions(-DCL_USE_DEPRECATED_OPENCL_1_1_APIS=1)
add_definitions(-DCL_USE_DEPRECATED_OPENCL_1_0_APIS=1)

# where to look first for cmake modules, before ${CMAKE_ROOT}/Modules/ is checked
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules)


# Support both VS2008 and VS2012.
set(BUILD_DIR "$ENV{ADRENO_DRIVER}/build")
if(MSVC90)
  set(VS_BUILD_DIR "${BUILD_DIR}/vs2008")
else(MSVC110)
  set(VS_BUILD_DIR "${BUILD_DIR}/vs2012")
endif(MSVC90)

#-----------------------------------------------------------
# Default Configurable Test Set
#-----------------------------------------------------------
set(D3D10_IS_SUPPORTED)
set(D3D11_IS_SUPPORTED)
set(GL_IS_SUPPORTED)


#-----------------------------------------------------------
# Tests prefix and suffix
#-----------------------------------------------------------
# Set the prefix and suffix for the generated executables
# For example, if you want the api executable to be test_conformance_api_12
# Set previx to "test_conformance_" and suffix to "_12"
set(CONFORMANCE_PREFIX "test_" )
set(CONFORMNACE_SUFFIX "" )

#-----------------------------------------------------------
# Vendor Customization
#-----------------------------------------------------------
#Vendor Customization File can be included here to provide a way to automatically
#build driver as a depencency of the conformance tests, or other such CMake customization
option(USE_VENDOR_CUSTOM_FILE "Use Vendor Customization File" OFF)
if(USE_VENDOR_CUSTOM_FILE)
    include(CMakeVendor.txt OPTIONAL)
endif(USE_VENDOR_CUSTOM_FILE)

#-----------------------------------------------------------
# Development options for OpenCL C++ tests
#-----------------------------------------------------------
# Use OpenCL C kernels instead of OpenCL C++ kernels
option(CLPP_DEVELOPMENT_USE_OPENCLC_KERNELS "Use OpenCL C kernels in OpenCL C++ tests" OFF)
if(CLPP_DEVELOPMENT_USE_OPENCLC_KERNELS)
  set(CLPP_DEVELOPMENT_OPTIONS ${CLPP_DEVELOPMENT_OPTIONS} -DCLPP_DEVELOPMENT_USE_OPENCLC_KERNELS)
endif(CLPP_DEVELOPMENT_USE_OPENCLC_KERNELS)
# Only check if OpenCL C++ kernels compile to SPIR-V
option(CLPP_DEVELOPMENT_ONLY_SPIRV_COMPILATION "Only check if OpenCL C++ kernels compile to SPIR-V" OFF)
if(CLPP_DEVELOPMENT_ONLY_SPIRV_COMPILATION)
  if(CLPP_DEVELOPMENT_USE_OPENCLC_KERNELS)
    message(FATAL_ERROR "Can't use OpenCL C kernels and compile to SPIR-V.")
  endif(CLPP_DEVELOPMENT_USE_OPENCLC_KERNELS)
  set(CLPP_DEVELOPMENT_OPTIONS ${CLPP_DEVELOPMENT_OPTIONS} -DCLPP_DEVELOPMENT_ONLY_SPIRV_COMPILATION)
endif(CLPP_DEVELOPMENT_ONLY_SPIRV_COMPILATION)
#
if(CLPP_DEVELOPMENT_OPTIONS)
  add_definitions(-DCLPP_DEVELOPMENT_OPTIONS)
  add_definitions(${CLPP_DEVELOPMENT_OPTIONS})
endif(CLPP_DEVELOPMENT_OPTIONS)

# Offline OpenCL C/C++ compiler provided by Khronos is the only supported
# offline compiler.
#
# Path to offline OpenCL C/C++ compiler provided by Khronos.
# See https://github.com/KhronosGroup/SPIR/ (spirv-1.1 branch or newer SPIR-V-ready
# branch should be used).
if(CL_OFFLINE_COMPILER)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DCL_OFFLINE_COMPILER=${CL_OFFLINE_COMPILER}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCL_OFFLINE_COMPILER=${CL_OFFLINE_COMPILER}")
    # Additional OpenCL C/C++ compiler option.
    if(CL_OFFLINE_COMPILER_OPTIONS)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DCL_OFFLINE_COMPILER_OPTIONS=${CL_OFFLINE_COMPILER_OPTIONS}")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCL_OFFLINE_COMPILER_OPTIONS=${CL_OFFLINE_COMPILER_OPTIONS}")
    endif(CL_OFFLINE_COMPILER_OPTIONS)
else(CL_OFFLINE_COMPILER)
    message(STATUS "OpenCL C/C++ compiler hasn't been found!")
    message(FATAL_ERROR "Pass path to OpenCL C/C++ compiler in CL_OFFLINE_COMPILER")
endif(CL_OFFLINE_COMPILER)

# CL_LIBCLCXX_DIR - path to dir with OpenCL C++ STL (libclcxx)
# CL_INCLUDE_DIR - path to dir with OpenCL headers
# CL_LIBCLCXX_DIR - path to dir with OpenCL library
if(CL_INCLUDE_DIR AND CL_LIB_DIR AND CL_LIBCLCXX_DIR)
    set(OPENCL_INCLUDE_DIR ${CL_INCLUDE_DIR})
    link_directories(${CL_LIB_DIR})
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DCL_LIBCLCXX_DIR=${CL_LIBCLCXX_DIR}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCL_LIBCLCXX_DIR=${CL_LIBCLCXX_DIR}")
else(CL_INCLUDE_DIR AND CL_LIB_DIR AND CL_LIBCLCXX_DIR)
    message(STATUS "OpenCL hasn't been found!")
    message(FATAL_ERROR "Either install OpenCL or pass -DCL_INCLUDE_DIR, -DCL_LIB_DIR and -DCL_LIBCLCXX_DIR")
endif(CL_INCLUDE_DIR AND CL_LIB_DIR AND CL_LIBCLCXX_DIR)

include(CheckFunctionExists)
include(CheckIncludeFiles)

if(CMAKE_COMPILER_IS_GNUCC OR "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
    # -msse -mfpmath=sse to force gcc to use sse for float math,
    # avoiding excess precision problems that cause tests like int2float
    # to falsely fail. -ffloat-store also works, but WG suggested
    # that sse would be better.
    if(CMAKE_ARM_COMPILER OR ANDROID)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -std=gnu99 -Wno-narrowing")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -std=gnu++11 -Wno-narrowing")
    else(CMAKE_ARM_COMPILER OR ANDROID)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -std=gnu99 -msse -mfpmath=sse -Wno-narrowing")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -msse -mfpmath=sse -std=gnu++11 -Wno-narrowing")
    endif(CMAKE_ARM_COMPILER OR ANDROID)
else()
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /D__SSE__")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /D__SSE__")
endif()

list(APPEND CLConform_LIBRARIES ${OPENCL_LIBRARIES})
if(ANDROID)
    list(APPEND CLConform_LIBRARIES m)
elseif(NOT WIN32)
    list(APPEND CLConform_LIBRARIES pthread)
endif(ANDROID)

if(APPLE)
    find_library(corefoundation CoreFoundation)
    find_library(iokit IOKit)
    list(APPEND CLConform_LIBRARIES ${corefoundation})
    list(APPEND CLConform_LIBRARIES ${iokit})
endif(APPLE)

include_directories(SYSTEM ${OPENCL_INCLUDE_DIR})
include_directories(${CLConform_SOURCE_DIR}/test_common/harness
                    ${CLConform_SOURCE_DIR}/test_common/gles
                    ${CLConform_SOURCE_DIR}/test_common/gl
                    ${CMAKE_CURRENT_SOURCE_DIR}/test_common/harness)

if(CMAKE_BUILD_TYPE STREQUAL "release")
    set (BUILD_FLAVOR "release")
elseif (CMAKE_BUILD_TYPE STREQUAL "debug")
    set (BUILD_FLAVOR "debug")
endif(CMAKE_BUILD_TYPE STREQUAL "release")


add_subdirectory(test_conformance)

set (PY_PATH   "${CLConform_SOURCE_DIR}/test_conformance/*.py")
set (CSV_PATH  "${CLConform_SOURCE_DIR}/test_conformance/*.csv")
# Support both VS2008 and VS2012.
set (DLL_FILES "${VS_BUILD_DIR}/Debug/*.dll")
set (DST_DIR   "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/Debug/")

if (WIN32)
    set (COPY "echo")
    add_custom_target(COPY_DLL${CONFORMANCE_SUFFIX} ALL
                      COMMAND ${COPY} "${DLL_FILES}" "${DST_DIR}"
                      COMMENT "Copying dll files.. ")
else (WIN32)
    set (COPY cp)
    add_custom_target(COPY_DLL${CONFORMANCE_SUFFIX})
endif(WIN32)

set_property(TARGET COPY_DLL${CONFORMANCE_SUFFIX} PROPERTY FOLDER "CONFORMANCE${CONFORMANCE_SUFFIX}")

if(WIN32)
  add_custom_target( COPY_FILES${CONFORMANCE_SUFFIX} ALL
                     COMMAND ${COPY} ${PY_PATH} ${DST_DIR}
                     COMMAND ${COPY} ${CSV_PATH} ${DST_DIR}
                     COMMAND ${COPY} ${DLL_FILES} ${DST_DIR}
                     COMMENT "Copying other files to output folder..." )
else(WIN32)
  add_custom_target( COPY_FILES${CONFORMANCE_SUFFIX} )
endif(WIN32)

# Copy required CL include directories into the build directory
# as required for the compiler testing.

# ... For running the compiler test on the command line.
file(COPY "${CLConform_SOURCE_DIR}/test_conformance/compiler/includeTestDirectory" DESTINATION ${DST_DIR})
file(COPY "${CLConform_SOURCE_DIR}/test_conformance/compiler/secondIncludeTestDirectory" DESTINATION ${DST_DIR})

# ... For running the compiler test with VisualStudio.
if(MSVC)
  file(COPY "${CLConform_SOURCE_DIR}/test_conformance/compiler/includeTestDirectory" DESTINATION "${CLConform_SOURCE_DIR}/build/test_conformance/compiler")
  file(COPY "${CLConform_SOURCE_DIR}/test_conformance/compiler/secondIncludeTestDirectory" DESTINATION "${CLConform_SOURCE_DIR}/build/test_conformance/compiler")
endif(MSVC)

set_property(TARGET COPY_FILES${CONFORMANCE_SUFFIX} PROPERTY FOLDER "CONFORMANCE${CONFORMANCE_SUFFIX}")
