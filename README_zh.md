[English](README.md) | **中文**

<p align="center" style="font-size: 36px; font-weight: bold; color: #797979;">sqlite3-cmake</p>

![CMake](https://img.shields.io/badge/CMake-3.14%2B-brightgreen)
![License](https://img.shields.io/badge/license-MIT-blue.svg)
![SQLite](https://img.shields.io/badge/SQLite-3.51.0-green.svg)
[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/tayne3/sqlite3-cmake)

一个为 [SQLite3](https://www.sqlite.org/) amalgamation 源码提供的 CMake 封装项目，旨在帮助你轻松地将 SQLite3 集成到自己的 CMake 项目中。

## 项目集成

#### 使用 [FetchContent](https://cmake.org/cmake/help/latest/module/FetchContent.html)

```cmake
cmake_minimum_required(VERSION 3.14)
project(example)

include(FetchContent)
FetchContent_Declare(sqlite3
    GIT_REPOSITORY https://github.com/tayne3/sqlite3-cmake
    GIT_TAG v3.50.4
)
FetchContent_MakeAvailable(sqlite3)

add_executable(example main.c)
target_link_libraries(example sqlite3::sqlite3)
```

#### 使用 [CPM.cmake](https://github.com/cpm-cmake/CPM.cmake)

使用自带的 SQLite3 源码:

```cmake
cmake_minimum_required(VERSION 3.14)
project(example)

if(NOT DEFINED ENV{CPM_SOURCE_CACHE})
 set(ENV{CPM_SOURCE_CACHE} ${CMAKE_SOURCE_DIR}/cmake_modules)
endif()
include(cmake/CPM.cmake)

CPMAddPackage("gh:tayne3/sqlite3-cmake@3.51.0")

add_executable(example main.c)
target_link_libraries(example sqlite3::sqlite3)
```

使用指定版本的 SQLite3 源码:

```cmake
cmake_minimum_required(VERSION 3.14)
project(example)

if(NOT DEFINED ENV{CPM_SOURCE_CACHE})
 set(ENV{CPM_SOURCE_CACHE} ${CMAKE_SOURCE_DIR}/cmake_modules)
endif()
include(cmake/CPM.cmake)

CPMAddPackage(
    NAME sqlite3
    URI "gh:tayne3/sqlite3-cmake@3.51.0"
    OPTIONS
    "SQLITE3_EXTERNAL_URL https://www.sqlite.org/2025/sqlite-amalgamation-3490100.zip"
    "SQLITE3_EXTERNAL_HASH SHA3_256=e7eb4cfb2d95626e782cfa748f534c74482f2c3c93f13ee828b9187ce05b2da7"
)

add_executable(example main.c)
target_link_libraries(example sqlite3::sqlite3)
```
