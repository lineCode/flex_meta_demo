﻿# see https://github.com/chromium/chromium/blob/9db0b5162b8648833c3a6b1af044cebd9d7d3855/base/BUILD.gn

cmake_minimum_required(VERSION 3.4)

# project() must be before checks, see https://stackoverflow.com/a/26437667/10904212
set(PROJECT_NAME "flex_meta_demo")
set(PROJECT_VERSION 1.0.0.1)
project(${PROJECT_NAME}
  VERSION ${PROJECT_VERSION}
  LANGUAGES CXX C)

set(LIB_NAME ${PROJECT_NAME})

option(ENABLE_TESTS "Enable tests" OFF)

set(BUILD_SHARED_LIBS
  TRUE CACHE BOOL
  "Use .so/.dll")

if(BUILD_SHARED_LIBS)
  set(CORE_LIB_TYPE SHARED)
else()
  set(CORE_LIB_TYPE STATIC)
  message(FATAL_ERROR "ONLY SHARED PLUGINS SUPPORTED FOR NOW")
endif(BUILD_SHARED_LIBS)

set(ENABLE_CLING TRUE CACHE BOOL "ENABLE_CLING")
message(STATUS "ENABLE_CLING=${ENABLE_CLING}")

set(ENABLE_CLANG_FROM_CONAN FALSE CACHE BOOL "ENABLE_CLANG_FROM_CONAN")
message(STATUS "ENABLE_CLANG_FROM_CONAN=${ENABLE_CLANG_FROM_CONAN}")

if(ENABLE_CLANG_FROM_CONAN AND ENABLE_CLING)
  message(FATAL_ERROR
    "don't use both ENABLE_CLING and ENABLE_CLANG_FROM_CONAN at the same time. cling already provides clang libtooling")
endif()

set(CUSTOM_PLUGINS
    "${CMAKE_CURRENT_SOURCE_DIR}/custom_plugins.cmake"
    CACHE STRING
    "Path to custom plugins")
message(STATUS "CUSTOM_PLUGINS=${CUSTOM_PLUGINS}")
if(EXISTS ${CUSTOM_PLUGINS})
  include(${CUSTOM_PLUGINS})
endif()

# --------------------------- conan configuration ------------------------------

# Note: FetchContent is available since CMake 3.11
# see https://cmake.org/cmake/help/git-master/module/FetchContent.html
# By default, everything is downloaded into your build directory
# Once CMake successfully downloads our external content, it sets two variables that can be used in CMakeLists.txt to locate the new data:
# 1 <resource_name>_SOURCE_DIR
#   specifies the location of the downloaded sources,
# 2 <resource_name>_BINARY_DIR
#   specifies where is the default build directory for the downloaded sources.
include(FetchContent)

set(FetchContent_cmake_utils "cmake_utils")
FetchContent_Declare(
  ${FetchContent_cmake_utils}
  PREFIX external_dependencies/${FetchContent_cmake_utils}
  GIT_REPOSITORY https://github.com/blockspacer/cmake_utils.git
  GIT_TAG origin/master
  # Disable warning about detached HEAD https://stackoverflow.com/a/36794768
  GIT_CONFIG        advice.detachedHead=false
  SOURCE_DIR        "${CMAKE_BINARY_DIR}/${FetchContent_cmake_utils}"
  BINARY_DIR        "${CMAKE_BINARY_DIR}/${FetchContent_cmake_utils}-build"
  CMAKE_ARGS        "-Wno-dev"
  CONFIGURE_COMMAND ""
  BUILD_COMMAND     ""
  INSTALL_COMMAND   ""
  TEST_COMMAND      ""
)
FetchContent_GetProperties(${FetchContent_cmake_utils})
if (NOT ${FetchContent_cmake_utils}_POPULATED)
  # Populate command can be used only once for every resource
  # during cmake configuration, which explains the conditional above.
  FetchContent_Populate(${FetchContent_cmake_utils})
endif()
message(STATUS "${FetchContent_cmake_utils}_SOURCE_DIR = ${${FetchContent_cmake_utils}_SOURCE_DIR}")
message(STATUS "${FetchContent_cmake_utils}_BINARY_DIR = ${${FetchContent_cmake_utils}_BINARY_DIR}")
include(${${FetchContent_cmake_utils}_SOURCE_DIR}/Utils.cmake)

set(FetchContent_conan_auto_install "conan_auto_install")
FetchContent_Declare(
  ${FetchContent_conan_auto_install}
  PREFIX external_dependencies/${FetchContent_conan_auto_install}
  GIT_REPOSITORY https://github.com/blockspacer/conan_auto_install.git
  GIT_TAG origin/master
  # Disable warning about detached HEAD https://stackoverflow.com/a/36794768
  GIT_CONFIG        advice.detachedHead=false
  SOURCE_DIR        "${CMAKE_BINARY_DIR}/${FetchContent_conan_auto_install}"
  BINARY_DIR        "${CMAKE_BINARY_DIR}/${FetchContent_conan_auto_install}-build"
  CMAKE_ARGS        "-Wno-dev"
  CONFIGURE_COMMAND ""
  BUILD_COMMAND     ""
  INSTALL_COMMAND   ""
  TEST_COMMAND      ""
)
FetchContent_GetProperties(${FetchContent_conan_auto_install})
if (NOT ${FetchContent_conan_auto_install}_POPULATED)
  # Populate command can be used only once for every resource
  # during cmake configuration, which explains the conditional above.
  FetchContent_Populate(${FetchContent_conan_auto_install})
endif()
message(STATUS "${FetchContent_conan_auto_install}_SOURCE_DIR = ${${FetchContent_conan_auto_install}_SOURCE_DIR}")
message(STATUS "${FetchContent_conan_auto_install}_BINARY_DIR = ${${FetchContent_conan_auto_install}_BINARY_DIR}")
include(${${FetchContent_conan_auto_install}_SOURCE_DIR}/conan-auto-install.cmake)
option(CONAN_AUTO_INSTALL "Let CMake call conan install automatically"
  #OFF # disable by default cause llvm requires long build with CONAN_LLVM_SINGLE_THREAD_BUILD=1
  ON
)
if (CONAN_AUTO_INSTALL)
  set(CONAN_PROFILE
      "clang"
      CACHE STRING "Conan profile to use during installation")
  if (NOT CMAKE_BUILD_TYPE MATCHES "Debug" )
    set(conan_build_type "Release")
  else()
    set(conan_build_type "Debug")
  endif()
  # No --build=missing cause llvm requires long build
  conan_auto_install(
    CONAN_OPTIONS "--profile=${CONAN_PROFILE} -s build_type=${conan_build_type} -s cling_conan:build_type=Release -o openssl:shared=True"
    #FORCE
  )
endif()

if(EXISTS "${CMAKE_CURRENT_BINARY_DIR}/conanbuildinfo.cmake")
  list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_BINARY_DIR}/)
  include(${CMAKE_CURRENT_BINARY_DIR}/conanbuildinfo.cmake)
  include(${CMAKE_CURRENT_BINARY_DIR}/conan_paths.cmake OPTIONAL)
  conan_basic_setup(
    # prevent conan_basic_setup from resetting cmake variables
    TARGETS
    KEEP_RPATHS
    # see https://github.com/conan-io/conan/issues/6012
    NO_OUTPUT_DIRS
    )
else()
  message (FATAL_ERROR "must use conan")
endif()

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules")

list(APPEND CMAKE_PROGRAM_PATH ${CONAN_BIN_DIRS})
# TODO: make better: link lib dirs
link_directories(${CONAN_LIB_DIRS})
include(HelperFlextool) # requires list(APPEND CMAKE_PROGRAM_PATH ${CONAN_BIN_DIRS})
set(flex_reflect_plugin
  ${CONAN_FLEX_REFLECT_PLUGIN_ROOT}/lib/flex_reflect_plugin${CMAKE_SHARED_LIBRARY_SUFFIX}
)
message(STATUS "flex_reflect_plugin=${flex_reflect_plugin}")
set(flex_meta_plugin
  ${CONAN_FLEX_META_PLUGIN_ROOT}/lib/flex_meta_plugin${CMAKE_SHARED_LIBRARY_SUFFIX}
)
message(STATUS "flex_meta_plugin=${flex_meta_plugin}")
set(cling_includes
  ${CONAN_CLING_CONAN_ROOT}/include
)
message(STATUS "cling_includes=${cling_includes}")
set(clang_includes
  ${CONAN_CLING_CONAN_ROOT}/lib/clang/5.0.0/include
)
message(STATUS "clang_includes=${clang_includes}")
set(flex_support_headers
  ${CONAN_FLEX_SUPPORT_HEADERS_ROOT}/include/flex/cling_preloader.inc
)
message(STATUS "flex_support_headers=${flex_support_headers}")
set(flexlib_headers
  ${CONAN_FLEXLIB_ROOT}/include
)
message(STATUS "flexlib_headers=${flexlib_headers}")
set(basis_headers
  ${CONAN_BASIS_ROOT}/include
)
message(STATUS "basis_headers=${basis_headers}")
set(chromium_base_headers
  ${CONAN_CHROMIUM_BASE_ROOT}/include
)
message(STATUS "chromium_base_headers=${chromium_base_headers}")
set(chromium_build_util_headers
  ${CONAN_CHROMIUM_BUILD_UTIL_ROOT}/include/chromium
)
message(STATUS "chromium_build_util_headers=${chromium_build_util_headers}")
set(flextool_input_files ${CMAKE_CURRENT_SOURCE_DIR}/src/main.cc)
set(flextool_outdir ${CMAKE_BINARY_DIR})
message(STATUS "flextool_outdir=${flextool_outdir}")
# requires set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
#execute_process(
#  COMMAND ${CMAKE_COMMAND} -E copy_if_different
#          ${CMAKE_BINARY_DIR}/compile_commands.json
#          ${CMAKE_CURRENT_SOURCE_DIR}
#)

# TODO: HACK: first run may produce errors
# second run must be fine if compile_commands.json generated fine
#
# requires existing compile_commands.json so
# must be run before build, but after configure step
execute_process(
  COMMAND ${flextool}
          --indir=${CMAKE_CURRENT_SOURCE_DIR}
          # outdir can contain `compile_commands.json`.
          # To create compile_commands.json
          # set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
          # or
          # set_target_properties(... ENABLE_EXPORTS 1)
          # see https://releases.llvm.org/9.0.0/tools/clang/docs/LibTooling.html
          # NOTE: Cling does not support compile_commands.json
          --outdir=${flextool_outdir}
          --load_plugin=${flex_reflect_plugin}
          --load_plugin=${flex_meta_plugin}
          --extra-arg=-I${cling_includes}
          --extra-arg=-I${clang_includes}
          # NOTE: Cling does not support compile_commands.json
          # so we must add headers used by `flex_support_headers` package
          # place into `flex_support_headers` includes that can be used by Cling
          --extra-arg=-I${chromium_base_headers}
          --extra-arg=-I${chromium_build_util_headers}
          --extra-arg=-I${basis_headers}
          --extra-arg=-I${flexlib_headers}
          --extra-arg=-Wno-undefined-inline
          # NOTE: we provide compile_commands.json to flextool
          # with information about project specific include dirs and defines
          # so you don't need to provide `-D` or `-I` arguments to flextool `by hand`
          ${flextool_extra_args}
          ${flextool_input_files}
          --cling_scripts=${flex_support_headers}
          #TIMEOUT 7200 # sec
  RESULT_VARIABLE retcode
  ERROR_VARIABLE _ERROR_VARIABLE)
if(NOT "${retcode}" STREQUAL "0")
  message(FATAL_ERROR "Bad exit status ${retcode} ${_ERROR_VARIABLE}")
endif()
message(STATUS "flextool output: ${retcode} ${_ERROR_VARIABLE}")

# Generate clang compilation database
# see https://stackoverflow.com/a/31086619/10904212
#set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

find_package(cmake_platform_detection REQUIRED)
run_cmake_platform_detection()

find_package(cmake_build_options REQUIRED)
setup_default_build_type(RELEASE)
setup_cmake_build_options(RELEASE DEBUG)

message(STATUS "Compiler ${CMAKE_CXX_COMPILER}, version: ${CMAKE_CXX_COMPILER_VERSION}")

set_project_version(0 0 1) # from Utils.cmake

check_cmake_build_type_selected() # from Utils.cmake

enable_colored_diagnostics() # from Utils.cmake

print_cmake_system_info() # from Utils.cmake

check_supported_os() # from Utils.cmake

# Keep symbols for JIT resolution
set(LLVM_NO_DEAD_STRIP 1)

if(NOT TARGET CONAN_PKG::chromium_base)
  message(FATAL_ERROR "Use chromium_base from conan")
endif()

if(NOT TARGET CONAN_PKG::clang_folly_conan)
  message(FATAL_ERROR "Use clang_folly_conan from conan")
endif()


if(NOT TARGET CONAN_PKG::chromium_build_util)
  message(FATAL_ERROR "Use chromium_build_util from conan")
endif()

find_package(chromium_build_util REQUIRED)
#
if(TARGET chromium_build_util::chromium_build_util-static)
  set(build_util_LIB "chromium_build_util::chromium_build_util-static")
else()
  message(FATAL_ERROR "not supported: using system provided chromium_build_util library")
endif()

find_package(chromium_base REQUIRED)
if(TARGET chromium_base::chromium_base-static)
  set(base_LIB chromium_base::chromium_base-static)
else()
  message(FATAL_ERROR "not supported: using system provided chromium_base library")
endif()

# see https://doc.magnum.graphics/corrade/corrade-cmake.html#corrade-cmake-subproject
find_package(Corrade REQUIRED PluginManager)

if(NOT TARGET CONAN_PKG::corrade)
  message(FATAL_ERROR "Use corrade from conan")
endif()

find_package( X11 REQUIRED )
message(STATUS "X11_LIBRARIES = ${X11_LIBRARIES}")

find_package( EXPAT REQUIRED )
message(STATUS "EXPAT_LIBRARIES = ${EXPAT_LIBRARIES}")

find_package( ZLIB REQUIRED )
message(STATUS "ZLIB_LIBRARIES = ${ZLIB_LIBRARIES}")

#add_library( boost_outcome INTERFACE )
#target_include_directories( boost_outcome SYSTEM INTERFACE "submodules/boost.outcome/include" )
#get_target_property (BOOST_OUTCOME_IMPORTED_LOCATION boost_outcome INTERFACE_INCLUDE_DIRECTORIES)
#message( STATUS "boost_outcome=${BOOST_OUTCOME_IMPORTED_LOCATION}" )
#
#add_library(microsoft_gsl INTERFACE)
#target_include_directories(microsoft_gsl SYSTEM INTERFACE "submodules/GSL/include")
#get_target_property (microsoft_gsl_IMPORTED_LOCATION microsoft_gsl INTERFACE_INCLUDE_DIRECTORIES)
#message( STATUS "microsoft_gsl=${microsoft_gsl_IMPORTED_LOCATION}" )

message(STATUS "CMAKE_DL_LIBS = ${CMAKE_DL_LIBS}")

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/)

set(flex_meta_demo_include_DIR "${CMAKE_CURRENT_SOURCE_DIR}/include/flex_meta_demo")
set(flex_meta_demo_src_DIR "${CMAKE_CURRENT_SOURCE_DIR}/src")

include(cmake/PlatformChecks)

include(cmake/CommonOptions)

include(cmake/ProjectFiles)

add_library(${LIB_NAME}-test-includes INTERFACE)

# $<INSTALL_INTERFACE:...> is exported using install(EXPORT)
# $<BUILD_INTERFACE:...> is exported using export(), or when the target is used by another target in the same buildsystem
macro(add_relative_include_dir TARGET VISIBILITY_BUILD VISIBILITY_INSTALL NEW_ELEM)
  target_include_directories(${TARGET}
    ${VISIBILITY_BUILD} "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/${NEW_ELEM}>"
    ${VISIBILITY_INSTALL} "$<INSTALL_INTERFACE:$<INSTALL_PREFIX>/${CMAKE_INSTALL_INCLUDEDIR}/${NEW_ELEM}>"
  )
  target_include_directories( ${LIB_NAME}-test-includes SYSTEM INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/${NEW_ELEM} )
endmacro(add_relative_include_dir)

# NOTE: cling does not support static libs, so we use SHARED
# https://github.com/root-project/cling/issues/280
# https://gitlab.kitware.com/cmake/cmake/issues/16473
add_executable(${LIB_NAME} #${CORE_LIB_TYPE}
  ${flex_meta_demo_SOURCES}
)

# This is the second signature of add_custom_command
# which adds a custom command to a target such as a library or executable.
# This is useful for performing an operation before or after building the target.
# The command becomes part of the target and
# will only execute when the target itself is built.
# If the target is already built, the command will not execute
add_custom_command(TARGET ${LIB_NAME}
    # On Visual Studio Generators, run before any other rules are executed within the target. On other generators, run just before PRE_LINK commands
    PRE_BUILD
    COMMAND ${flextool}
            --indir=${CMAKE_CURRENT_SOURCE_DIR}
            # outdir can contain `compile_commands.json`.
            # To create compile_commands.json
            # set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
            # or
            # set_target_properties(... ENABLE_EXPORTS 1)
            # see https://releases.llvm.org/9.0.0/tools/clang/docs/LibTooling.html
            # NOTE: Cling does not support compile_commands.json
            --outdir=${flextool_outdir}
            --load_plugin=${flex_reflect_plugin}
            --load_plugin=${flex_meta_plugin}
            --extra-arg=-I${cling_includes}
            --extra-arg=-I${clang_includes}
            # NOTE: Cling does not support compile_commands.json
            # so we must add headers used by `flex_support_headers` package
            # place into `flex_support_headers` includes that can be used by Cling
            --extra-arg=-I${chromium_base_headers}
            --extra-arg=-I${chromium_build_util_headers}
            --extra-arg=-I${basis_headers}
            --extra-arg=-I${flexlib_headers}
            --extra-arg=-Wno-undefined-inline
            # NOTE: we provide compile_commands.json to flextool
            # with information about project specific include dirs and defines
            # so you don't need to provide `-D` or `-I` arguments to flextool `by hand`
            ${flextool_extra_args}
            ${flextool_input_files}
            --cling_scripts=${flex_support_headers}
    COMMENT "running ${flextool}"
    VERBATIM # to support \t for example
)

add_relative_include_dir(${LIB_NAME} PUBLIC PUBLIC "include")
add_relative_include_dir(${LIB_NAME} PUBLIC PUBLIC "src")
# NOTE: we provide compile_commands.json to flextool
# with information about project specific include dirs and defines
# so you don't need to provide `-D` or `-I` arguments to flextool `by hand`
add_relative_include_dir(${LIB_NAME} PUBLIC PUBLIC "src/project_specific_headers")

#target_include_directories(${LIB_NAME} PRIVATE
#  ${CMAKE_CURRENT_SOURCE_DIR}/src/project_specific_header
#)

list(APPEND USED_3DPARTY_LIBS
  ${base_LIB}
  ${build_util_LIB}
)

if(NOT TARGET CONAN_PKG::flexlib)
  message(FATAL_ERROR "Use flexlib from conan")
endif()

list(APPEND USED_3DPARTY_LIBS
  CONAN_PKG::flexlib
  CONAN_PKG::basis
  CONAN_PKG::clang_folly_conan
  CONAN_PKG::boost
  CONAN_PKG::double-conversion
  CONAN_PKG::glog
  CONAN_PKG::lz4
  CONAN_PKG::lzma
  CONAN_PKG::zstd
  CONAN_PKG::snappy
  CONAN_PKG::libsodium
  CONAN_PKG::libdwarf
  CONAN_PKG::bzip2
  CONAN_PKG::gflags
  CONAN_PKG::libunwind
  CONAN_PKG::libelf
  CONAN_PKG::xz_utils
  CONAN_PKG::corrade
  Corrade::PluginManager
  ${USED_BOOST_LIBS}
  #boost_outcome
  #microsoft_gsl
)

set_property(TARGET ${LIB_NAME} PROPERTY CXX_STANDARD 17)

if(TARGET_EMSCRIPTEN)
  # use PROPERTY CXX_STANDARD 17
else()
  target_compile_features(${LIB_NAME}
    PUBLIC cxx_auto_type
    PRIVATE cxx_variadic_templates)
endif()

set(USED_SYSTEM_LIBS
    Threads::Threads # pthread, https://cmake.org/cmake/help/v3.13/module/FindThreads.html
    ${X11_LIBRARIES} # https://cmake.org/cmake/help/v3.13/module/FindX11.html
    ${CMAKE_DL_LIBS} # https://cmake.org/cmake/help/v3.13/variable/CMAKE_DL_LIBS.html
    EXPAT::EXPAT # https://cmake.org/cmake/help/v3.13/module/FindEXPAT.html
    ZLIB::ZLIB # https://cmake.org/cmake/help/v3.13/module/FindZLIB.html
               # Custom libs
    stdc++fs # C++17 std::filesystem
    # @note: Order matters https://stackoverflow.com/a/10269201/10904212
    ${LIBIBERTY_LIBRARY} # used by folly
    ${DOUBLE_CONVERSION_LIBRARY} # used by folly
    ${LIBEVENT_LIB} # used by folly
    ${LZ4_LIBRARY} # used by folly
    ${LIBUNWIND_LIBRARIES} # used by folly
    ${LIBLZMA_LIBRARIES} # used by folly
    CACHE INTERNAL "USED_SYSTEM_LIBS")

target_link_libraries(${LIB_NAME} PUBLIC#PRIVATE
  ${USED_3DPARTY_LIBS}
  ${USED_SYSTEM_LIBS}
)

set(DEBUG_LIBRARY_SUFFIX "")
set_target_properties(${LIB_NAME}
  PROPERTIES
    ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}" # TODO: /lib
    LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}" # TODO: /lib
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}" # TODO: /bin
    OUTPUT_NAME "${LIB_NAME}$<$<CONFIG:Debug>:${DEBUG_LIBRARY_SUFFIX}>"
    # Plugins don't have any prefix (e.g. 'lib' on Linux)
    PREFIX ""
    ENABLE_EXPORTS 1
    CXX_STANDARD 17
    CXX_EXTENSIONS OFF
    CMAKE_CXX_STANDARD_REQUIRED ON
)

# POSITION_INDEPENDENT_CODE for -fPIC
set_property(TARGET ${LIB_NAME} PROPERTY POSITION_INDEPENDENT_CODE ON)

if(ENABLE_CLING)
  find_package(Cling REQUIRED)

  list(APPEND CLING_DEFINITIONS CLING_IS_ON=1)
  target_link_libraries(${LIB_NAME} PUBLIC#PRIVATE
    CONAN_PKG::cling_conan
  )

  get_target_property (cling_conan_IMPORTED_LOCATION CONAN_PKG::cling_conan INTERFACE_INCLUDE_DIRECTORIES)
  message( STATUS "cling_conan=${cling_conan_IMPORTED_LOCATION}" )
  target_include_directories( ${LIB_NAME} PUBLIC ${cling_conan_IMPORTED_LOCATION} )

  if(MSVC)
    set_target_properties(${LIB_NAME} PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS 1)
    set_property(
      TARGET ${LIB_NAME}
      APPEND_STRING
      PROPERTY LINK_FLAGS
               "/EXPORT:?setValueNoAlloc@internal@runtime@cling@@YAXPEAX00D_K@Z
                /EXPORT:?setValueNoAlloc@internal@runtime@cling@@YAXPEAX00DM@Z
                /EXPORT:cling_runtime_internal_throwIfInvalidPointer")
  endif()

  target_compile_definitions(${LIB_NAME} PUBLIC CLING_IS_ON=1)
endif(ENABLE_CLING)

if(ENABLE_CLANG_FROM_CONAN)
  target_link_libraries( ${LIB_NAME} PUBLIC#PRIVATE
    CONAN_PKG::libclang
    CONAN_PKG::clang_tooling
    CONAN_PKG::clang_tooling_core
    CONAN_PKG::llvm_support
  )
endif(ENABLE_CLANG_FROM_CONAN)

target_compile_options(${LIB_NAME} PRIVATE
  $<$<CXX_COMPILER_ID:GNU>:-Wall>)

target_compile_options(${LIB_NAME} PRIVATE
  -fno-rtti)

target_compile_definitions(${LIB_NAME} PRIVATE
  ${flex_meta_demo_PRIVATE_DEFINES}
  # https://stackoverflow.com/a/30877725
  BOOST_SYSTEM_NO_DEPRECATED BOOST_ERROR_CODE_HEADER_ONLY
  # see https://github.com/mosra/corrade/blob/af9d4216f07307a2dff471664eed1e50e180568b/modules/UseCorrade.cmake#L568
  CORRADE_DYNAMIC_PLUGIN=1
)

target_compile_definitions(${LIB_NAME} PUBLIC
  ${flex_meta_demo_PUBLIC_DEFINES}
  ${CLING_DEFINITIONS}
)

# install and export steps
include(cmake/Exports)

if(ENABLE_TESTS)
  add_subdirectory( tests )
endif()
