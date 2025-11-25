cmake_minimum_required(VERSION 3.14 FATAL_ERROR)
project(
  sqlite3
  VERSION 3.50.4
  DESCRIPTION "SQLite3 CMake wrapper"
  LANGUAGES C
)

include(CMakePackageConfigHelpers)
include(CheckCSourceRuns)
include(CheckFunctionExists)
include(CheckIncludeFiles)
include(CheckLibraryExists)
include(CheckSymbolExists)
include(CheckTypeSize)
include(FetchContent)
include(GNUInstallDirs)

include(${CMAKE_CURRENT_LIST_DIR}/cmake/ProjectIsTopLevel.cmake)
include(${CMAKE_CURRENT_LIST_DIR}/cmake/CPM.cmake)

if(PROJECT_IS_TOP_LEVEL)
  get_property(isMultiConfig GLOBAL PROPERTY GENERATOR_IS_MULTI_CONFIG)
  if(NOT isMultiConfig
     AND NOT CMAKE_BUILD_TYPE
     AND NOT CMAKE_CONFIGURATION_TYPES
  )
    message(STATUS "Setting build type to 'Release' as none was specified.")
    set(CMAKE_BUILD_TYPE
        "Release"
        CACHE STRING "Choose the type of build." FORCE
    )
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
  endif()
  if(NOT DEFINED ENV{CPM_SOURCE_CACHE})
    set(ENV{CPM_SOURCE_CACHE} ${CMAKE_SOURCE_DIR}/cmake_modules)
  endif()
endif()

set(SQLITE3_EXTERNAL_URL
    ""
    CACHE STRING "Download external URL (empty to use local source)"
)
set(SQLITE3_EXTERNAL_HASH
    ""
    CACHE STRING "Download external hash (empty to use local source)"
)
option(SQLITE3_BUILD_SHARED_LIBS "Build shared libraries" OFF)
option(SQLITE3_ENABLE_THREADSAFE "Build a thread-safe library" OFF)
option(SQLITE3_ENABLE_DYNAMIC_EXTENSIONS "Support loadable extensions" OFF)
option(SQLITE3_ENABLE_MATH "Enable SQL math functions" ON)
option(SQLITE3_ENABLE_FTS3 "Include FTS3 support" OFF)
option(SQLITE3_ENABLE_FTS4 "Include FTS4 support" OFF)
option(SQLITE3_ENABLE_FTS5 "Include FTS5 support" ON)
option(SQLITE3_ENABLE_RTREE "Include R-tree support" ON)
option(SQLITE3_ENABLE_SESSION "Enable the session extension" OFF)
option(SQLITE3_OMIT_DEPRECATED "Omit deprecated features" ON)

if(SQLITE3_BUILD_SHARED_LIBS)
  set(BUILD_SHARED_LIBS ON)
else()
  set(BUILD_SHARED_LIBS OFF)
endif()
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED OFF)
set(CMAKE_C_EXTENSIONS ON)
set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Use local source or download SQLite source from URL
if(NOT SQLITE3_EXTERNAL_URL)
  set(SQLITE3_SOURCE_FILE ${PROJECT_SOURCE_DIR}/src/sqlite3.c)
  set(SQLITE3_HEADER_DIR ${PROJECT_SOURCE_DIR}/src)
else()
  CPMAddPackage(
    NAME
    sqlite3-external
    URL
    ${SQLITE3_EXTERNAL_URL}
    URL_HASH
    ${SQLITE3_EXTERNAL_HASH}
    DOWNLOAD_ONLY
  )
  set(SQLITE3_SOURCE_FILE ${sqlite3-external_SOURCE_DIR}/sqlite3.c)
  set(SQLITE3_HEADER_DIR ${sqlite3-external_SOURCE_DIR})
endif()

# Extract SQLite version from source
file(READ ${SQLITE3_SOURCE_FILE} SQLITE3_SOURCE)
string(REGEX MATCH "SQLITE_VERSION[ \t]+\"([^\"]+)\"" _ ${SQLITE3_SOURCE})
set(SQLITE3_VERSION "${CMAKE_MATCH_1}")
message(STATUS "SQLite version: ${SQLITE3_VERSION}")

add_library(sqlite3 ${SQLITE3_SOURCE_FILE})
add_library(sqlite3::sqlite3 ALIAS sqlite3)
target_include_directories(
  sqlite3 PUBLIC $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}> $<BUILD_INTERFACE:${SQLITE3_HEADER_DIR}>
)

install(
  TARGETS sqlite3
  EXPORT sqlite3-targets
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  INCLUDES
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)
install(FILES ${SQLITE3_HEADER_DIR}/sqlite3.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

function(check_and_define items check_function)
  foreach(item ${${items}})
    string(TOUPPER ${item} item_upper)
    string(MAKE_C_IDENTIFIER ${item_upper} item_upper)
    if(${check_function} STREQUAL "check_include_files")
      check_include_files(${item} ${item_upper})
    elseif(${check_function} STREQUAL "check_type_size")
      check_type_size(${item} ${item_upper})
    elseif(${check_function} STREQUAL "check_function_exists")
      check_function_exists(${item} ${item_upper})
    endif()
    if(${item_upper})
      target_compile_definitions(sqlite3 PRIVATE "HAVE_${item_upper}=1")
    endif()
  endforeach()
  unset(${items})
endfunction()

set(HEADERS "dlfcn.h" "stdint.h" "inttypes.h" "utime.h")
check_and_define(HEADERS check_include_files)

set(TYPES int8_t uint8_t int16_t uint16_t uint32_t)
check_and_define(TYPES check_type_size)

set(FUNCS
    fdatasync
    usleep
    fullfsync
    localtime_r
    gmtime_r
    strerror_r
    posix_fallocate
    nanosleep
)
check_and_define(FUNCS check_function_exists)

if(STRERROR_R)
  target_compile_definitions(sqlite3 PRIVATE HAVE_STRERROR_R=1 HAVE_DECL_STRERROR_R=1)
  check_c_source_runs(
    "
    #include <string.h>
    int main() {
        return (strerror_r(0, (char*)0, 0) == (char*)0);
    }
    "
    HAVE_GNU_STRERROR_R
  )
  if(HAVE_GNU_STRERROR_R)
    target_compile_definitions(sqlite3 PRIVATE STRERROR_R_CHAR_P=1)
  endif()
endif()

if(NOT SQLITE3_ENABLE_DYNAMIC_EXTENSIONS)
  set(HAVE_DLOPEN OFF)
elseif(WIN32)
  set(HAVE_DLOPEN ON)
else()
  check_library_exists(dl dlopen "" HAVE_DLOPEN)
  if(NOT HAVE_DLOPEN)
    message(FATAL_ERROR "No dlopen() in dl")
  endif()
endif()

if(NOT SQLITE3_ENABLE_MATH)
  set(HAVE_CEIL_IN_M OFF)
elseif(WIN32)
  set(HAVE_CEIL_IN_M ON)
else()
  check_library_exists(m ceil "" HAVE_CEIL_IN_M)
  if(NOT HAVE_CEIL_IN_M)
    message(FATAL_ERROR "No ceil() in m")
  endif()
endif()

if(NOT SQLITE3_ENABLE_FTS5)
  set(HAVE_LOG_IN_M OFF)
elseif(WIN32)
  set(HAVE_LOG_IN_M ON)
else()
  check_library_exists(m log "" HAVE_LOG_IN_M)
  if(NOT HAVE_LOG_IN_M)
    message(FATAL_ERROR "No log() in m")
  endif()
endif()

target_compile_definitions(
  sqlite3
  PRIVATE $<IF:$<BOOL:${SQLITE3_ENABLE_THREADSAFE}>,SQLITE_THREADSAFE=1
          _REENTRANT=1,SQLITE_THREADSAFE=0>
          $<$<NOT:$<BOOL:${SQLITE3_ENABLE_DYNAMIC_EXTENSIONS}>>:SQLITE_OMIT_LOAD_EXTENSION=1>
          $<$<BOOL:${SQLITE3_ENABLE_MATH}>:SQLITE_ENABLE_MATH_FUNCTIONS>
          $<$<AND:$<BOOL:${SQLITE3_ENABLE_FTS3}>,$<NOT:$<BOOL:${SQLITE3_ENABLE_FTS4}>>>:SQLITE_ENABLE_FTS3>
          $<$<BOOL:${SQLITE3_ENABLE_FTS4}>:SQLITE_ENABLE_FTS4>
          $<$<BOOL:${SQLITE3_ENABLE_FTS5}>:SQLITE_ENABLE_FTS5>
          $<$<BOOL:${SQLITE3_ENABLE_RTREE}>:SQLITE_ENABLE_RTREE
          SQLITE_ENABLE_GEOPOLY>
          $<$<BOOL:${SQLITE3_ENABLE_SESSION}>:SQLITE_ENABLE_SESSION
          SQLITE_ENABLE_PREUPDATE_HOOK>
          $<$<BOOL:${SQLITE3_OMIT_DEPRECATED}>:SQLITE_OMIT_DEPRECATED>
          $<$<BOOL:${HAVE_DLOPEN}>:HAVE_DLOPEN=1>
          $<$<CONFIG:Debug>:SQLITE_DEBUG
          SQLITE_ENABLE_SELECTTRACE
          SQLITE_ENABLE_WHERETRACE>
          SQLITE_ENABLE_EXPLAIN_COMMENTS
          SQLITE_DQS=0
          SQLITE_ENABLE_DBPAGE_VTAB
          SQLITE_ENABLE_STMTVTAB
          SQLITE_ENABLE_DBSTAT_VTAB
)

target_link_libraries(
  sqlite3 $<$<BOOL:${HAVE_DLOPEN}>:dl> $<$<OR:$<BOOL:${HAVE_CEIL_IN_M}>,$<BOOL:${HAVE_LOG_IN_M}>>:m>
)

# Generate and install package configuration files
configure_package_config_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/sqlite3-config.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/sqlite3-config.cmake
  INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/sqlite3
)

write_basic_package_version_file(
  ${CMAKE_CURRENT_BINARY_DIR}/sqlite3-config-version.cmake
  VERSION ${SQLITE3_VERSION}
  COMPATIBILITY SameMajorVersion
)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/sqlite3-config.cmake ${CMAKE_CURRENT_BINARY_DIR}/sqlite3-config-version.cmake
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/sqlite3
)

install(
  EXPORT sqlite3-targets
  FILE sqlite3-targets.cmake
  NAMESPACE sqlite3::
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/sqlite3
)
