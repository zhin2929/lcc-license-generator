cmake_minimum_required(VERSION 3.6 FATAL_ERROR)

SET(CMAKE_DISABLE_SOURCE_CHANGES ON)
SET(CMAKE_DISABLE_IN_SOURCE_BUILD ON)
SET(LICENSECC_SHORT_LICENSE "BSD Software License")

# 设置 CMake 策略，使用新的 Boost 查找方式
if(POLICY CMP0167)
  cmake_policy(SET CMP0167 NEW)
endif()

# 启用现代MSVC运行时库选择（CMake 3.15+）
if(POLICY CMP0091)
  cmake_policy(SET CMP0091 NEW)
endif()

cmake_policy(SET CMP0048 NEW)
project (lccgen
	VERSION 2.0.0
	DESCRIPTION "License generator for licensecc" 
	LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake/")

include(utilities) #import module utilities.cmake

#Not sure about this. My system doesn't support binfmt misc
IF( ( CMAKE_HOST_SYSTEM_NAME STREQUAL "Linux" ) AND CMAKE_CROSSCOMPILING AND NOT DEFINED CMAKE_CROSSCOMPILING_EMULATOR )
	SET(CMAKE_CROSSCOMPILING_EMULATOR "wine")	
ENDIF()

SET ( OPENSSL_USE_STATIC_LIBS ON )

# lccgen: 强制使用 vcpkg 的静态 OpenSSL（x64-windows-static triplet，与 Boost 同路径）
if(MSVC)
  # 从 CMAKE_TOOLCHAIN_FILE 推导 vcpkg 根目录
  if(CMAKE_TOOLCHAIN_FILE AND CMAKE_TOOLCHAIN_FILE MATCHES "vcpkg.cmake")
    get_filename_component(VCPKG_ROOT_TEMP "${CMAKE_TOOLCHAIN_FILE}" DIRECTORY)
    get_filename_component(VCPKG_ROOT_TEMP "${VCPKG_ROOT_TEMP}" DIRECTORY)
    get_filename_component(VCPKG_ROOT_TEMP "${VCPKG_ROOT_TEMP}" DIRECTORY)

    set(OPENSSL_ROOT_DIR "${VCPKG_ROOT_TEMP}/installed/x64-windows-static")
    message(STATUS "lccgen: Using static OpenSSL from ${OPENSSL_ROOT_DIR}")
  elseif(EXISTS "E:/vcpkg/installed/x64-windows-static")
    set(OPENSSL_ROOT_DIR "E:/vcpkg/installed/x64-windows-static")
    message(STATUS "lccgen: Using static OpenSSL from ${OPENSSL_ROOT_DIR}")
  endif()
endif()

find_package(OpenSSL COMPONENTS Crypto QUIET)
IF(OPENSSL_FOUND)
	add_definitions(-DHAS_OPENSSL)
	include_directories(${OPENSSL_INCLUDE_DIR})
	MESSAGE(STATUS "Found openssl version ${OPENSSL_VERSION} - generator")
ENDIF(OPENSSL_FOUND)

if(UNIX) #this is true for all the linux systems but not for cross compiling "linux->windows"
	#openssl required for linux
	IF(NOT OPENSSL_FOUND)
		MESSAGE(SEND_ERROR "Openssl not found - generator - required in Linux")
	ENDIF(NOT OPENSSL_FOUND)
	IF(${STATIC_RUNTIME})
		set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--whole-archive -static-libgcc -static-libstdc++ -static")
		unset(CMAKE_DL_LIBS)
	ENDIF(${STATIC_RUNTIME})
	
    find_package(Threads)
	if(THREADS_HAVE_PTHREAD_ARG)
  		list(APPEND EXTERNAL_LIBS "-pthread")
	else(THREADS_HAVE_PTHREAD_ARG)
		if(CMAKE_THREAD_LIBS_INIT)
  			 list(APPEND EXTERNAL_LIBS "${CMAKE_THREAD_LIBS_INIT}")
		endif()
	endif(THREADS_HAVE_PTHREAD_ARG)
	
	#Zlib required when openssl version < 1.0.2k (centos 7)
	IF(OPENSSL_VERSION VERSION_LESS_EQUAL 1.0.2)
		SET ( ZLIB_USE_STATIC_LIBS ON ) 
		find_package(ZLIB REQUIRED)
		list(APPEND EXTERNAL_LIBS ${ZLIB_LIBRARIES})
		MESSAGE(STATUS "Found zlib version ${ZLIB_VERSION_STRING} - generator")
	ENDIF(OPENSSL_VERSION VERSION_LESS_EQUAL 1.0.2)

	find_program( MEMORYCHECK_COMMAND valgrind )
	set( MEMORYCHECK_COMMAND_OPTIONS "--trace-children=yes --leak-check=full" ) 
	set(pkg_config_dest "lib/cmake/${PROJECT_NAME}")
else(UNIX) #windows and linux crosscompiling for windows target
	IF(NOT OPENSSL_FOUND)
		MESSAGE(STATUS "Openssl not found - generator - configuring libary without it.")
	ENDIF(NOT OPENSSL_FOUND)
	if(MSVC)
		include(target_arch)
		target_architecture( TARGET_ARCHITECTURE )
		message(STATUS "architecture detected: ${TARGET_ARCHITECTURE}")

		# lccgen 作为独立工具，强制使用静态CRT（CMake 3.15+方式，自动应用到所有目标）
		set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>" CACHE STRING "MSVC runtime library" FORCE)
		message(STATUS "lccgen: Using static CRT (/MT for Release, /MTd for Debug)")

		add_definitions("/D _CRT_SECURE_NO_WARNINGS")
		list(APPEND EXTERNAL_LIBS "bcrypt" "crypt32" "ws2_32" "iphlpapi")
	else(MSVC)
	    if(MINGW)
	        list(APPEND EXTERNAL_LIBS "-lcrypt32 -lbcrypt -lws2_32 -liphlpapi")
	        SET(CMAKE_EXE_LINKER_FLAGS "-static")
	    endif(MINGW)
	endif(MSVC)
	set(pkg_config_dest "cmake/${PROJECT_NAME}")
endif(UNIX)

#bug in cmake win32 - can't find boost compiled with mingw 
if(WIN32 AND "x${CMAKE_CXX_COMPILER_ARCHITECTURE_ID}" STREQUAL "x")
    message(WARNING "WIN32 compiler does not specify CMAKE_CXX_COMPILER_ARCHITECTURE_ID -- filling in manually")
    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
        set(CMAKE_CXX_COMPILER_ARCHITECTURE_ID "x64")
    else()
        set(CMAKE_CXX_COMPILER_ARCHITECTURE_ID "x86")
    endif()
    message(STATUS "Compiler architecture: ${CMAKE_CXX_COMPILER_ARCHITECTURE_ID}")
endif()

SET( Boost_USE_STATIC_LIBS ON )  # 静态链接 Boost 库本身

# 配置 Boost 运行时匹配
if(MSVC)
    # lccgen 为独立工具，使用静态 CRT (/MT) 编译的 Boost 库（与 x64-windows-static 匹配）
    SET(Boost_USE_STATIC_RUNTIME ON)

    # 禁用 Boost 自动链接机制
    add_definitions(-DBOOST_ALL_NO_LIB)
    # 显示链接诊断信息
    add_definitions(-DBOOST_LIB_DIAGNOSTIC)
endif()

# lccgen: 强制使用 vcpkg 的静态 Boost 库（x64-windows-static triplet，包含 /MT CRT）
if(MSVC)
  # 从 CMAKE_TOOLCHAIN_FILE 推导 vcpkg 根目录（重用之前的变量）
  if(VCPKG_ROOT_TEMP)
    set(BOOST_ROOT "${VCPKG_ROOT_TEMP}/installed/x64-windows-static")
    set(Boost_NO_SYSTEM_PATHS ON)
    message(STATUS "lccgen: Derived vcpkg root from toolchain: ${VCPKG_ROOT_TEMP}")
    message(STATUS "lccgen: Using static Boost from ${BOOST_ROOT}")
  elseif(CMAKE_TOOLCHAIN_FILE AND CMAKE_TOOLCHAIN_FILE MATCHES "vcpkg.cmake")
    get_filename_component(VCPKG_ROOT "${CMAKE_TOOLCHAIN_FILE}" DIRECTORY)
    get_filename_component(VCPKG_ROOT "${VCPKG_ROOT}" DIRECTORY)
    get_filename_component(VCPKG_ROOT "${VCPKG_ROOT}" DIRECTORY)

    set(BOOST_ROOT "${VCPKG_ROOT}/installed/x64-windows-static")
    set(Boost_NO_SYSTEM_PATHS ON)
    message(STATUS "lccgen: Derived vcpkg root from toolchain: ${VCPKG_ROOT}")
    message(STATUS "lccgen: Using static Boost from ${BOOST_ROOT}")
  elseif(DEFINED ENV{VCPKG_ROOT})
    # 备用：使用环境变量
    set(BOOST_ROOT "$ENV{VCPKG_ROOT}/installed/x64-windows-static")
    set(Boost_NO_SYSTEM_PATHS ON)
    message(STATUS "lccgen: Using vcpkg static Boost from ${BOOST_ROOT}")
  elseif(EXISTS "E:/vcpkg/installed/x64-windows-static")
    # 备用：硬编码路径
    set(BOOST_ROOT "E:/vcpkg/installed/x64-windows-static")
    set(Boost_NO_SYSTEM_PATHS ON)
    message(STATUS "lccgen: Using vcpkg static Boost from ${BOOST_ROOT}")
  endif()
endif()

find_package(Boost REQUIRED COMPONENTS date_time filesystem program_options system unit_test_framework)

if(Boost_FOUND)
    message(STATUS "Found Boost (static libs): ${Boost_VERSION}")
endif()

include_directories(${Boost_INCLUDE_DIRS}) 

configure_file (
	"src/build_properties.h.in" 
	"${CMAKE_BINARY_DIR}/include/build_properties.h"
)

message( STATUS "CXX compiler              : " ${CMAKE_CXX_COMPILER})
message( STATUS "CXX compiler flags        : " ${CMAKE_CXX_FLAGS})
message( STATUS "CXX compiler flags debug  : " ${CMAKE_CXX_FLAGS_DEBUG})
message( STATUS "CXX compiler flags release: " ${CMAKE_CXX_FLAGS_RELEASE})
if(CMAKE_BUILD_TYPE)
  message( STATUS "Build type                : " ${CMAKE_BUILD_TYPE})
endif(CMAKE_BUILD_TYPE)
message( STATUS "Install prefix            : " ${CMAKE_INSTALL_PREFIX})
#include (build directory)/include to find build_properties.h
message( STATUS "External libs             : " ${EXTERNAL_LIBS})
include_directories(${CMAKE_BINARY_DIR}/include)
add_subdirectory("src/base_lib")
add_subdirectory("src/license_generator")

INCLUDE(CTest)
IF(BUILD_TESTING)
    SET(BUILDNAME "${BUILDNAME}" CACHE STRING "Name of build on the dashboard")
	MARK_AS_ADVANCED(BUILDNAME)
	add_subdirectory("test")
ENDIF(BUILD_TESTING)

set(CPACK_GENERATOR "DEB;TBZ2;RPM")
set(CPACK_PACKAGE_NAME "lcc-generator")
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "open license manager Team")
set(CPACK_PACKAGING_INSTALL_PREFIX ${CMAKE_INSTALL_PREFIX})
set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME}-${PROJECT_VERSION}.${CMAKE_SYSTEM_PROCESSOR}")
set(CPACK_OUTPUT_FILE_PREFIX dist)
set(CPACK_RESOURCE_FILE_LICENSE "${PROJECT_SOURCE_DIR}/LICENSE")
include(CPack)

