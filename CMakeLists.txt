# CMake build file - use CMake 3.2 or later to build WDT and its dependencies.
#
# mkdir build; cd build; cmake .. -DBUILD_TESTING=on; make -j
# omit -DBUILD_TESTING=on if you don't want the extra dependencies for
# testing (but testing is good !)
# (at fb:
# cd local; mkdir wdt_build; cd wdt_build
# cmake31 ~/fbcode/wdt -DFOLLY_SOURCE_DIR=$HOME/fbcode -DBUILD_TESTING=on
# make -j
# )
#
#  Copyright (c) 2014-present, Facebook, Inc.
#  All rights reserved.
#
#  This source code is licensed under the BSD-style license found in the
#  LICENSE file in the root directory of this source tree.
#

cmake_minimum_required(VERSION 3.2)

# There is no C per se in WDT but if you use CXX only here many checks fail
# Version is Major.Minor.YYMMDDX for up to 10 releases per day (X from 0 to 9)
# Minor currently is also the protocol version - has to match with Protocol.cpp
project("WDT" LANGUAGES C CXX VERSION 1.32.1910230)

# WDT uses C++17 features per fdbc5432230290f86ff8ad89ab52d5b7fef232b4
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED on)

# somehow 'option' for this doesn't seeem to work/I don't know how to make it
set(BUILD_SHARED_LIBS off CACHE BOOL "build shared libs")

# CMake default behavior should be to set rpath when needed (non system install)
# it's not so let's set this for now:
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")

set(WDT_USE_SYSTEM_FOLLY Off CACHE BOOL "Use folly library from system (default off)")

# Optimized by default
# TODO: This doesn't seem to work / sets default to "" instead of Release...
# set(CMAKE_BUILD_TYPE Release CACHE String "build type")
# So hardcoding for now:
#set(CMAKE_BUILD_TYPE Debug)
set(CMAKE_BUILD_TYPE Release)
set(CMAKE_CXX_FLAGS "-msse4.2 -mpclmul")
#set(CMAKE_CXX_FLAGS "-msse4.2 -mpclmul -Wextra -Wsign-compare -Wunused-variable")
#set(CMAKE_CXX_FLAGS "-msse4.2 -mpclmul -Wextra -Wsign-compare -Wunused-variable -Wconversion -Wsign-conversion")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "_bin/wdt")

if (NOT WDT_USE_SYSTEM_FOLLY)
  # Check that we have the Folly source tree
  set(FOLLY_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../folly" CACHE PATH
      "Folly source tree (folly/Conv.h should be reachable from there")
  # Check for folly - TODO: this doesn't work well for relative paths
  # (because of relative to build dir vs relative to source tree for -I)
  if(NOT EXISTS "${FOLLY_SOURCE_DIR}/folly/Conv.h")
    MESSAGE(FATAL_ERROR "${FOLLY_SOURCE_DIR}/folly/Conv.h not found
  Fix using:
  (in a sister directory of the wdt source tree - same level:)
  git clone https://github.com/facebook/folly.git
  or change FOLLY_SOURCE_DIR (use ccmake or -DFOLLY_SOURCE_DIR=...)
  or change WDT_USE_SYSTEM_FOLLY (use ccmake or -DWDT_USE_SYSTEM_FOLLY=...)
  ")
  endif()


  # The part of folly that isn't pure .h and we use:
  set (FOLLY_CPP_SRC
  "${FOLLY_SOURCE_DIR}/folly/Conv.cpp"
  "${FOLLY_SOURCE_DIR}/folly/Demangle.cpp"
  "${FOLLY_SOURCE_DIR}/folly/lang/CString.cpp"
  "${FOLLY_SOURCE_DIR}/folly/lang/ToAscii.cpp"
  "${FOLLY_SOURCE_DIR}/folly/hash/Checksum.cpp"
  "${FOLLY_SOURCE_DIR}/folly/hash/detail/ChecksumDetail.cpp"
  "${FOLLY_SOURCE_DIR}/folly/hash/detail/Crc32cDetail.cpp"
  "${FOLLY_SOURCE_DIR}/folly/hash/detail/Crc32CombineDetail.cpp"
  "${FOLLY_SOURCE_DIR}/folly/ScopeGuard.cpp"
  "${FOLLY_SOURCE_DIR}/folly/detail/SplitStringSimd.cpp"
  "${FOLLY_SOURCE_DIR}/folly/detail/TrapOnAvx512.cpp"
  "${FOLLY_SOURCE_DIR}/folly/external/fast-crc32/sse_crc32c_v8s3x3.cpp"
  )
endif()

# WDT's library proper - comes from: ls -1 *.cpp | grep -iv test
add_library(wdt_min STATIC
util/WdtSocket.cpp
util/ClientSocket.cpp
util/EncryptionUtils.cpp
util/DirectorySourceQueue.cpp
ErrorCodes.cpp
util/FileByteSource.cpp
util/FileCreator.cpp
Protocol.cpp
WdtThread.cpp
util/ThreadsController.cpp
ReceiverThread.cpp
Receiver.cpp
WdtTransferRequest.cpp
Reporting.cpp
util/ThreadTransferHistory.cpp
SenderThread.cpp
Sender.cpp
util/ServerSocket.cpp
Throttler.cpp
WdtOptions.cpp
util/FileWriter.cpp
util/TransferLogManager.cpp
util/SerializationUtil.cpp
util/Stats.cpp
WdtBase.cpp
WdtResourceController.cpp
util/CommonImpl.cpp
)
# Source files that depend on gflags and provide flags -> options init
set (WDT_FLAGS_RELATED_SRC
util/WdtFlags.cpp
Wdt.cpp
)
add_library(wdt STATIC ${WDT_FLAGS_RELATED_SRC})

set_property(TARGET wdt PROPERTY VERSION ${PROJECT_VERSION})
set_property(TARGET wdt_min PROPERTY VERSION ${PROJECT_VERSION})
target_link_libraries(wdt wdt_min)



# Folly uses boost system and filesystem
#set(Boost_USE_STATIC_LIBS on)
find_package(Boost COMPONENTS system filesystem REQUIRED)
include_directories(${Boost_INCLUDE_DIRS})

# We use std:: threads
find_package(Threads) # this will set ${CMAKE_THREAD_LIBS_INIT} (ie pthreads)


# double-conversion
find_path(DOUBLECONV_INCLUDE_DIR double-conversion/double-conversion.h)
find_library(DOUBLECONV_LIBRARY double-conversion)
# Glog
find_path(GLOG_INCLUDE_DIR glog/logging.h)
find_library(GLOG_LIBRARY glog)
# Gflags
find_path(GFLAGS_INCLUDE_DIR gflags/gflags.h)
find_library(GFLAGS_LIBRARY gflags)
# GFLAGS_NAMESPACE is defined in most releases but not in all
include(CheckCXXSymbolExists)
list(APPEND CMAKE_REQUIRED_INCLUDES "${GFLAGS_INCLUDE_DIR}/")
list(APPEND CMAKE_REQUIRED_LIBRARIES "${GFLAGS_LIBRARY}")
check_cxx_symbol_exists(GFLAGS_NAMESPACE "gflags/gflags.h" CMAKE_GFLAGS_NAMESPACE)
if (NOT CMAKE_GFLAGS_NAMESPACE)
  check_cxx_symbol_exists(gflags::GetArgv "gflags/gflags.h" CMAKE_GFLAGS_GFLAGS_GETARGV)
  if (CMAKE_GFLAGS_GFLAGS_GETARGV)
    add_definitions("-DGFLAGS_NAMESPACE=gflags")
  else()
    check_cxx_symbol_exists(google::GetArgv "gflags/gflags.h" CMAKE_GFLAGS_GOOGLE_GETARGV)
    if (CMAKE_GFLAGS_GOOGLE_GETARGV)
      add_definitions("-DGFLAGS_NAMESPACE=google")
    endif()
  endif()
endif()
list(REMOVE_ITEM CMAKE_REQUIRED_INCLUDES "${GFLAGS_INCLUDE_DIR}/")
list(REMOVE_ITEM CMAKE_REQUIRED_LIBRARIES "${GFLAGS_LIBRARY}")
# OpenSSL's crypto lib
find_package(OpenSSL REQUIRED)
include_directories(${OPENSSL_INCLUDE_DIR})
# System Folly
if (WDT_USE_SYSTEM_FOLLY)
  find_path(FOLLY_INCLUDE_DIR folly/Conv.h)
  find_library(FOLLY_LIBRARY folly)
else()
  set(FOLLY_LIBRARY folly4wdt)
  set(FOLLY_INCLUDE_DIR "${FOLLY_SOURCE_DIR}")
endif()

# You can also add jemalloc to the list if you have it/want it
target_link_libraries(wdt_min
  ${FOLLY_LIBRARY}
  ${GLOG_LIBRARY}
  ${GFLAGS_LIBRARY}
  ${Boost_LIBRARIES}
  ${DOUBLECONV_LIBRARY}
  ${OPENSSL_CRYPTO_LIBRARY}
  ${CMAKE_THREAD_LIBS_INIT} # Must be last to avoid link errors
)

# What we need to build the part of folly we use:

include(CheckIncludeFileCXX)
include(CheckFunctionExists)
include(CheckLibraryExists)
include(CheckCXXSourceCompiles)
# For WDT itself:
check_function_exists(posix_fallocate HAS_POSIX_FALLOCATE)
check_function_exists(sync_file_range HAS_SYNC_FILE_RANGE)
check_function_exists(posix_memalign HAS_POSIX_MEMALIGN)
check_function_exists(posix_fadvise HAS_POSIX_FADVISE)
# C based check (which fail with the c++ setting thereafter...)
check_function_exists(clock_gettime FOLLY_HAVE_CLOCK_GETTIME)
# was: check_library_exists(rt clock_gettime "" FOLLY_HAVE_CLOCK_GETTIME)
set(SAVE_CMRL ${CMAKE_REQUIRED_LIBRARIES}) #globals are evil/ugly
set(CMAKE_REQUIRED_LIBRARIES ${CMAKE_THREAD_LIBS_INIT})
check_function_exists(pthread_atfork FOLLY_HAVE_PTHREAD_ATFORK)
set(CMAKE_REQUIRED_LIBRARIES ${SAVE_CMRL}) #globals are evil/ugly
# Needed until Cmake issue #15361 is addressed
# All the check_function_exists should be before this line
# or else they will not work'

set(CMAKE_REQUIRED_DEFINITIONS ${CMAKE_CXX11_STANDARD_COMPILE_OPTION})
check_include_file_cxx(linux/sockios.h WDT_HAS_SOCKIOS_H)
#check_function_exists(clock_gettime FOLLY_HAVE_CLOCK_GETTIME)
check_cxx_source_compiles(
"extern \"C\" void cmkcheckweak() __attribute__((weak));
int main(int argc, char** argv) {
  return (cmkcheckweak) ? 1 : 0 ;
}" FOLLY_HAVE_WEAK_SYMBOLS)
# Now record all this :
# Folly's:
configure_file(build/folly-config.h.in folly/folly-config.h)
# Wdt's config/version
configure_file(WdtConfig.h.in wdt/WdtConfig.h)

if (NOT WDT_USE_SYSTEM_FOLLY)
  # Malloc stuff  tied to not supporting weaksympbols
  if (NOT FOLLY_HAVE_WEAK_SYMBOLS)
    list(APPEND FOLLY_CPP_SRC "${FOLLY_SOURCE_DIR}/folly/memory/detail/MallocImpl.cpp")
    message(STATUS "no weak symbols, adding MallocImpl to folly src")
  endif()

  add_library(folly4wdt STATIC ${FOLLY_CPP_SRC})
  target_link_libraries(folly4wdt ${GLOG_LIBRARY} ${DOUBLECONV_LIBRARY})
endif()

# Order is important - inside fb we want the above
# folly-config.h to be picked up instead of the fbcode one
include_directories(${CMAKE_CURRENT_BINARY_DIR})
include_directories(${FOLLY_INCLUDE_DIR})
include_directories(${DOUBLECONV_INCLUDE_DIR})
include_directories(${GLOG_INCLUDE_DIR})
include_directories(${GFLAGS_INCLUDE_DIR})
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/..)

add_executable(wdtbin wdtCmdLine.cpp ${WDT_FLAGS_RELATED_SRC})
set_target_properties(wdtbin PROPERTIES COMPILE_DEFINITIONS "STANDALONE_APP")

target_link_libraries(wdtbin wdt_min)

### Install rules
set_target_properties(wdtbin PROPERTIES RUNTIME_OUTPUT_NAME "wdt")
install(TARGETS wdtbin wdt wdt_min
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
  )

if (NOT WDT_USE_SYSTEM_FOLLY)
  install(TARGETS folly4wdt
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    )
endif()

### Install header files

# Find the . files in the root directory

file(GLOB headers "*.h")
foreach(header ${headers})
  install(FILES ${header} DESTINATION include/wdt)
endforeach()

# Install the .h files in the util with directory structure maintained

install(DIRECTORY "${PROJECT_SOURCE_DIR}/util" DESTINATION include/wdt
        FILES_MATCHING PATTERN "*.h")

# wcp script
install(PROGRAMS test/wcp.sh DESTINATION bin RENAME wcp)


### Everything below is about testing (and not required to create wdt/wdt)

if (BUILD_TESTING)

  enable_testing()
  find_package(GTest CONFIG REQUIRED)
  if(NOT GTest_FOUND)
    include(ExternalProject)

    # GTest
    #  from https://github.com/google/googletest/tree/master/googletest
    configure_file(CMakeLists.txt.in googletest-download/CMakeLists.txt)
    execute_process(COMMAND ${CMAKE_COMMAND} -G "${CMAKE_GENERATOR}" .
      RESULT_VARIABLE result
      WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/googletest-download )
    if(result)
      message(FATAL_ERROR "CMake step for googletest failed: ${result}")
    endif()
    execute_process(COMMAND ${CMAKE_COMMAND} --build .
      RESULT_VARIABLE result
      WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/googletest-download )
    if(result)
      message(FATAL_ERROR "Build step for googletest failed: ${result}")
    endif()

    # Prevent overriding the parent project's compiler/linker
    # settings on Windows
    set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)

    # Add googletest directly to our build. This defines
    # the gtest and gtest_main targets.
    add_subdirectory(${CMAKE_CURRENT_BINARY_DIR}/googletest-src
                     ${CMAKE_CURRENT_BINARY_DIR}/googletest-build
                     EXCLUDE_FROM_ALL)
  endif()

  # Extra code that we use in tests
  add_library(wdt4tests_min
    test/TestCommon.cpp
  )
  target_link_libraries(wdt4tests_min
    gtest
    wdt_min
  )

  add_library(wdt4tests
    ${WDT_FLAGS_RELATED_SRC}
  )
  target_link_libraries(wdt4tests wdt4tests_min)

  # TODO: make a macro/function to add tests in 1 line instead of 3


  # WDT testing/benchmarking code
  add_library(wdtbenchlib
    bench/Bigram.cpp
  )

  target_link_libraries(wdtbenchlib
    ${GLOG_LIBRARY}
    ${GFLAGS_LIBRARY}
  )

  add_library(wdtbenchtestslib
    bench/WdtGenTestUtils.cpp
  )
  target_link_libraries(wdtbenchtestslib
    gtest
    wdtbenchlib
    ${CMAKE_THREAD_LIBS_INIT} # Must be last to avoid link errors
  )

  add_executable(wdt_gen_files bench/wdtGenFiles.cpp)
  target_link_libraries(wdt_gen_files wdtbenchlib
    ${CMAKE_THREAD_LIBS_INIT} # Must be last to avoid link errors
  )
  set_target_properties(wdt_gen_files PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY "_bin/wdt/bench/")

  add_executable(wdt_gen_stats bench/wdtStats.cpp)
  target_link_libraries(wdt_gen_stats wdtbenchlib)

  add_executable(wdt_gen_test bench/wdtGenTest.cpp)
  target_link_libraries(wdt_gen_test wdtbenchtestslib)
  add_test(NAME AllTestsInGenTest COMMAND wdt_gen_test)

  # Regular tests

  add_executable(protocol_test test/ProtocolTest.cpp)
  target_link_libraries(protocol_test wdt4tests)
  add_test(NAME AllTestsInProtocolTest COMMAND protocol_test)

  add_executable(test_encdeci64_func test/test_encdeci64_func.cpp)
  target_link_libraries(test_encdeci64_func wdt4tests)
  add_test(NAME test_encdeci64_func COMMAND test_encdeci64_func)

  add_executable(test_stats test/Stats_test.cpp)
  target_link_libraries(test_stats wdt4tests)
  add_test(NAME test_stats COMMAND test_stats)

  add_executable(histogram test/Histogram.cpp)
  target_link_libraries(histogram wdt_min)

  add_executable(resource_controller_test  test/WdtResourceControllerTest.cpp)
  target_link_libraries(resource_controller_test wdt4tests)
  add_test(NAME ResourceControllerTests COMMAND resource_controller_test)

  add_executable(wdt_url_test  test/WdtUrlTest.cpp)
  target_link_libraries(wdt_url_test wdt4tests)
  add_test(NAME WdtUrlTests COMMAND wdt_url_test)

  add_executable(wdt_misc_tests  test/WdtMiscTests.cpp)
  target_link_libraries(wdt_misc_tests wdt4tests)
  add_test(NAME WdtMiscTests COMMAND wdt_misc_tests)

  add_executable(wdt_fd_test  test/FdTest.cpp)
  target_link_libraries(wdt_fd_test wdt4tests)
  add_test(NAME WdtFdTests COMMAND wdt_fd_test)

  add_executable(encryption_test  test/EncryptionTest.cpp)
  target_link_libraries(encryption_test wdt4tests)
  add_test(NAME EncryptionTests COMMAND encryption_test)

  add_executable(file_reader_test  test/FileReaderTest.cpp)
  target_link_libraries(file_reader_test wdt4tests)
  add_test(NAME FileReaderTests COMMAND file_reader_test)

  add_executable(option_type_test_long_flags test/OptionTypeTest.cpp)
  target_link_libraries(option_type_test_long_flags wdt4tests)

  add_executable(option_type_test_short_flags test/OptionTypeTest.cpp
                                              ${WDT_FLAGS_RELATED_SRC}
                                              )
  set_target_properties(option_type_test_short_flags PROPERTIES
    COMPILE_DEFINITIONS "STANDALONE_APP"
    RUNTIME_OUTPUT_DIRECTORY "_bin/wdt/short_flags/")

  target_link_libraries(option_type_test_short_flags wdt4tests_min)

  add_test(NAME WdtRandGenTest COMMAND
    "${CMAKE_CURRENT_SOURCE_DIR}/test/wdt_rand_gen_test.sh")

  add_test(NAME WdtBasicE2E COMMAND
    "${CMAKE_CURRENT_SOURCE_DIR}/test/wdt_e2e_simple_test.sh")

# Doesn't work on a mac:
#  add_test(NAME WdtStdinManifestAndUrl COMMAND
#    "${CMAKE_CURRENT_SOURCE_DIR}/test/wdt_stdin_test.sh")

  add_test(NAME WdtLockFailFast COMMAND
    "${CMAKE_CURRENT_SOURCE_DIR}/test/wdt_lock_failfast.sh")

  add_test(NAME WdtBasicE2Exfs COMMAND
    "${CMAKE_CURRENT_SOURCE_DIR}/test/wdt_e2e_xfs_test.sh")

  add_test(NAME WdtOptionsTypeTests COMMAND
    "${CMAKE_CURRENT_SOURCE_DIR}/test/wdt_option_type_test.sh")

  add_test(NAME hostname_override_test COMMAND
    "${CMAKE_CURRENT_SOURCE_DIR}/test/hostname_override_test.py")

  add_test(NAME WdtPortBlockTest COMMAND
    "${CMAKE_CURRENT_SOURCE_DIR}/test/wdt_port_block_test.py")

  add_test(NAME WdtProtocolNegotiationTest COMMAND
    "${CMAKE_CURRENT_SOURCE_DIR}/test/wdt_protocol_negotiation_test.py")

  add_test(NAME WdtSimpleOdirectTest COMMAND
    "${CMAKE_CURRENT_SOURCE_DIR}/test/wdt_e2e_simple_test.sh" -o true)

  add_test(NAME WdtFileListTest COMMAND
    "${CMAKE_CURRENT_SOURCE_DIR}/test/wdt_file_list_test.py")

  add_test(NAME WdtOverwriteTest COMMAND
    "${CMAKE_CURRENT_SOURCE_DIR}/test/wdt_overwrite_test.py")

  add_test(NAME WdtBadServerTest COMMAND
    "${CMAKE_CURRENT_SOURCE_DIR}/test/wdt_bad_server_test.py")

  add_test(NAME ReceiverThrottlerRefCountTest COMMAND
    "${CMAKE_CURRENT_SOURCE_DIR}/test/receiver_throttler_ref_count_test.sh")

  add_test(NAME WdtLongRunningTest COMMAND
    "${CMAKE_CURRENT_SOURCE_DIR}/test/wdt_long_running_test.py")

endif(BUILD_TESTING)
