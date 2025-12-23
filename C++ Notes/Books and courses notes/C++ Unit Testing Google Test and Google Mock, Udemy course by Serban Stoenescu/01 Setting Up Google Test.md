## Setting with CMake
- Option 1: using [CPM](https://github.com/cpm-cmake/CPM.cmake):
```cmake
set(CPM_DOWNLOAD_LOCATION "${CMAKE_BINARY_DIR}/cmake/CPM.cmake")  
  
if (NOT EXISTS ${CPM_DOWNLOAD_LOCATION})  
    message(STATUS "Downloading CPM.cmake to ${CPM_DOWNLOAD_LOCATION}")  
    file(DOWNLOAD  
            https://github.com/cpm-cmake/CPM.cmake/releases/latest/download/get_cpm.cmake  
            ${CPM_DOWNLOAD_LOCATION}  
    )  
endif ()  
  
include(${CPM_DOWNLOAD_LOCATION})  
CPMAddPackage("gh:google/googletest#v1.17.0")  
  
add_executable(UnitTest unit_tests/someLibTest.cpp)  
target_link_libraries(UnitTest someLib GTest::gtest_main GTest::gmock_main)
```
+ Option 2: using `FetchContent`
```cmake 
# ====== Before 3.30 
include(FetchContent)  
FetchContent_Declare(gtest  
        GIT_REPOSITORY https://github.com/google/googletest  
        GIT_TAG        v1.17.0  
)  
FetchContent_GetProperties(gtest)  
if (NOT gtest_POPULATED)  
    FetchContent_Populate(gtest)  
    add_subdirectory(${gtest_SOURCE_DIR} ${gtest_BINARY_DIR})  
endif ()  
  
add_executable(UnitTest unit_tests/someLibTest.cpp)  
target_link_libraries(UnitTest someLib GTest::gtest_main GTest::gmock_main)

# ===== In more modern CMake
include(FetchContent)  
FetchContent_Declare(gtest  
        GIT_REPOSITORY https://github.com/google/googletest  
        GIT_TAG        v1.17.0  
)  
FetchContent_MakeAvailable(gtest)  
  
add_executable(UnitTest unit_tests/someLibTest.cpp)  
target_link_libraries(UnitTest someLib GTest::gtest_main GTest::gmock_main)
```

Also might be useful to enable CTest integration  
```cmake 
enable_testing()  
include(GoogleTest)  
gtest_discover_tests(UnitTest)
```

## Simple project
Usually when testing some parts of application (unit), that unit is part of some library, which is linked to executable main application. For unit testing such library we link it again test executable and inside test body launch function which is under the test 
```cmake 
cmake_minimum_required(VERSION 3.30)  
project(app VERSION 1.0.0 LANGUAGES C CXX)  
  
add_executable(app src/main.cpp)  
include_directories(src)  
  
add_library(someLib STATIC src/someLib.cpp)  
target_link_libraries(app someLib)  
  
include(cmake/CPM.cmake)  
CPMAddPackage("gh:google/googletest#v1.17.0")  
  
add_executable(UnitTest unit_tests/someLibTest.cpp)  
target_link_libraries(UnitTest someLib GTest::gtest_main GTest::gmock_main)  
  
enable_testing()  
include(GoogleTest)  
gtest_discover_tests(UnitTest)
```
For example, functions that we going to test is 
```cpp title:someLib.h
#pragma once  
  
int sum(int a, int b);
```
```cpp title:someLib.cpp
#include "someLib.h"  
  
int sum(int a, int b) {  
    return a + b;  
}
```

Test for this function will looks like 
```cpp:someLibTest.cpp
#include "someLib.h"  
#include <gtest/gtest.h>  
  
TEST(TestSuiteSample, TestSample) {  
    auto res = sum(2, 4);  
    ASSERT_EQ(6, res);  
}  

// Optional, actually CMake target gtest_main have similar main function.  
int main(int argc, char **argv) {  
    testing::InitGoogleTest(&argc, argv);  
    return RUN_ALL_TESTS();  
}
```