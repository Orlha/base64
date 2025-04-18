cmake_minimum_required(VERSION 3.16)
project(base64)

option(BASE64_ENABLE_TESTING "Build test files." OFF)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE RelWithDebInfo)
endif()

# Warnings compiler flags
if(MSVC)
  add_compile_options(/W4)
else()
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# Optionally enable clang-tidy
find_program(CLANG_TIDY_EXE NAMES clang-tidy PATHS /opt/homebrew/opt/llvm/bin/)
if(NOT CLANG_TIDY_EXE)
  message(STATUS "clang-tidy not found. Skipping corresponding checks.")
else()
  set(CMAKE_CXX_CLANG_TIDY 
    ${CLANG_TIDY_EXE};
    -header-filter=.*;
    -checks=-*,portability-*,bugprone-*,-bugprone-easily-swappable-parameters,-bugprone-sizeof-expression,clang-analyzer-*;
  )
  message(STATUS "Found clang-tidy: ${CLANG_TIDY_EXE}.")
endif()

# Simplify the use of ASan
option(USE_ASAN "Activate ASan compiler/linker options" OFF)
if(USE_ASAN)
  if(MSVC)
    add_compile_options(/fsanitize=address)
    add_link_options(/fsanitize=address)
    set(CMAKE_MSVC_DEBUG_INFORMATION_FORMAT "$<IF:$<AND:$<C_COMPILER_ID:MSVC>,$<CXX_COMPILER_ID:MSVC>>,$<$<CONFIG:Debug,RelWithDebInfo>:EditAndContinue>,$<$<CONFIG:Debug,RelWithDebInfo>:ProgramDatabase>>")
  else()
    add_compile_options(-fsanitize=address -fno-omit-frame-pointer)
    add_link_options(-fsanitize=address)
  endif()
endif()

add_library(base64 INTERFACE)
target_include_directories(base64 INTERFACE include)

if (BASE64_ENABLE_TESTING)
  add_executable(roundtrip_test test/roundtrip_test.cpp)
  target_link_libraries(roundtrip_test PRIVATE base64)

  enable_testing()
  add_test(NAME roundtrip_test COMMAND roundtrip_test)

  # Add some more tests
  include(FetchContent)
  if(${CMAKE_CXX_BYTE_ORDER} MATCHES BIG_ENDIAN)
    set(FETCHCONTENT_QUIET FALSE)
  endif()
  FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG        b514bdc898e2951020cbdca1304b75f5950d1f59
    GIT_PROGRESS   TRUE
    SYSTEM
  )
  # For Windows: Prevent overriding the parent project's compiler/linker settings
  set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
  FetchContent_MakeAvailable(googletest)

  set_target_properties(gtest PROPERTIES CXX_CLANG_TIDY "")
  set_target_properties(gtest_main PROPERTIES CXX_CLANG_TIDY "")
  set_target_properties(gmock PROPERTIES CXX_CLANG_TIDY "")
  set_target_properties(gmock_main PROPERTIES CXX_CLANG_TIDY "")

  add_executable(base64_tests test/base64_tests.cpp)
  target_link_libraries(base64_tests PRIVATE base64)
  target_link_libraries(base64_tests PRIVATE GTest::gtest GTest::gtest_main)
  add_test(NAME base64_tests COMMAND base64_tests)

  add_executable(modp_b64_tests test/modp_b64_tests.cpp)
  target_link_libraries(modp_b64_tests PRIVATE base64)
  target_link_libraries(modp_b64_tests PRIVATE GTest::gtest GTest::gtest_main)
  add_test(NAME modp_b64_tests COMMAND modp_b64_tests)
endif()

