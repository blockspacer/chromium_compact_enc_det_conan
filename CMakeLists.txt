# see https://github.com/chromium/chromium/blob/9db0b5162b8648833c3a6b1af044cebd9d7d3855/base/BUILD.gn

cmake_minimum_required(VERSION 3.4)

# project() must be before checks, see https://stackoverflow.com/a/26437667/10904212
set(PROJECT_NAME "chromium_compact_enc_det")
set(PROJECT_VERSION 1.0.0.1)
project(${PROJECT_NAME}
  VERSION ${PROJECT_VERSION}
  LANGUAGES CXX C)

set(STATIC_LIB_NAME ${PROJECT_NAME}-static)

option(COMPILE_WITH_LLVM_TOOLS
  "Enable clang from llvm_tools (conan package)" OFF)

# see https://github.com/Ericsson/codechecker/blob/master/tools/report-converter/README.md#undefined-behaviour-sanitizer
# NOTE: Compile with -g and -fno-omit-frame-pointer
# to get proper debug information in your binary.
# NOTE: Run your program with environment variable UBSAN_OPTIONS=print_stacktrace=1.
# see https://github.com/google/sanitizers/wiki/SanitizerCommonFlags
option(ENABLE_UBSAN
  "Enable Undefined Behaviour Sanitizer" OFF)

# see https://github.com/google/sanitizers/wiki/AddressSanitizerLeakSanitizer
# see https://github.com/Ericsson/codechecker/blob/master/tools/report-converter/README.md#address-sanitizer
# NOTE: Compile with -g and -fno-omit-frame-pointer
# to get proper debug information in your binary.
# NOTE: use ASAN_OPTIONS=detect_leaks=1 LSAN_OPTIONS=suppressions=suppr.txt
# NOTE: You need the ASAN_OPTIONS=symbolize=1
# to turn on resolving addresses in object code
# to source code line numbers and filenames.
# This option is implicit for Clang but it won't do any harm.
# see https://github.com/google/sanitizers/wiki/SanitizerCommonFlags
option(ENABLE_ASAN
  "Enable Address Sanitizer" OFF)

# see https://github.com/Ericsson/codechecker/blob/master/tools/report-converter/README.md#memory-sanitizer
# NOTE: Compile with -g and -fno-omit-frame-pointer
# to get proper debug information in your binary.
option(ENABLE_MSAN
  "Enable Memory Sanitizer" OFF)

# see https://github.com/Ericsson/codechecker/blob/master/tools/report-converter/README.md#thread-sanitizer
# NOTE: Compile with -g
# to get proper debug information in your binary.
option(ENABLE_TSAN
  "Enable Thread Sanitizer" OFF)

if(COMPILE_WITH_LLVM_TOOLS)
  message(STATUS
    "Using COMPILE_WITH_LLVM_TOOLS")
  # do not check compile in conanbuildinfo
  # cause we will switch to other compiler after conan install
  set(CONAN_DISABLE_CHECK_COMPILER ON)
else(COMPILE_WITH_LLVM_TOOLS)
  message(STATUS
    "Disabled COMPILE_WITH_LLVM_TOOLS")
endif(COMPILE_WITH_LLVM_TOOLS)

# --------------------------- conan configuration ------------------------------

# Conan package manager List of packages used by the project is container in
# conanfile.py
option(CONAN_AUTO_INSTALL "Let CMake call conan install automatically" OFF)
if (CONAN_AUTO_INSTALL)
  set(CONAN_PROFILE
      "default"
      CACHE STRING "Conan profile to use during installation")
  include(cmake/conan-auto-install.cmake)
  conan_auto_install(${CONAN_PROFILE})
endif()

if(EXISTS "${CMAKE_BINARY_DIR}/conanbuildinfo.cmake")
  list(APPEND CMAKE_MODULE_PATH ${CMAKE_BINARY_DIR}/)
  include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
  include(${CMAKE_BINARY_DIR}/conan_paths.cmake OPTIONAL)
  conan_basic_setup(
    # prevent conan_basic_setup from resetting cmake variables
    TARGETS
    KEEP_RPATHS
    # see https://github.com/conan-io/conan/issues/6012
    NO_OUTPUT_DIRS)
else()
  message (FATAL_ERROR "must use conan")
endif()

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/)

if(ENABLE_MSAN OR ENABLE_TSAN OR ENABLE_ASAN OR ENABLE_UBSAN)
  find_package(cmake_sanitizers REQUIRED)
endif()

find_package(cmake_helper_utils REQUIRED)

# prefer ASCII for folder names
force_latin_paths() # from cmake_helper_utils (conan package)

# out dirs (CMAKE_*_OUTPUT_DIRECTORY) must be not empty
validate_out_dirs() # from cmake_helper_utils (conan package)

# In-source builds not allowed
validate_out_source_build(WARNING) # from cmake_helper_utils (conan package)

if(ENABLE_MSAN)
  add_msan_flags()
endif(ENABLE_MSAN)

if(ENABLE_TSAN)
  add_tsan_flags()
endif(ENABLE_TSAN)

if(ENABLE_ASAN)
  add_asan_flags()
endif(ENABLE_ASAN)

if(ENABLE_UBSAN)
  add_ubsan_flags()
endif(ENABLE_UBSAN)

if(COMPILE_WITH_LLVM_TOOLS)
  message(STATUS
    "Using COMPILE_WITH_LLVM_TOOLS")
  # force change CMAKE_*_COMPILER and CMAKE_LINKER to clang from conan
  compile_with_llvm_tools() # from cmake_helper_utils (conan package)
else(COMPILE_WITH_LLVM_TOOLS)
  message(STATUS
    "Disabled COMPILE_WITH_LLVM_TOOLS")
endif(COMPILE_WITH_LLVM_TOOLS)

find_package(cmake_platform_detection REQUIRED)
run_cmake_platform_detection()
# NOTE: perform platform checks before everything else
include(cmake/PlatformChecks)

find_package(cmake_build_options REQUIRED)
setup_default_build_type(RELEASE)
setup_cmake_build_options(RELEASE DEBUG)

set(compact_enc_det_DIR "${CMAKE_CURRENT_SOURCE_DIR}/chromium/third_party/ced/src/")

if (NOT EXISTS "${compact_enc_det_DIR}compact_enc_det/compact_enc_det.h")
  message(FATAL_ERROR "Could not find ced source code.")
endif()

include(cmake/CommonOptions)

option(CED_BUILD_SHARED_LIBS "Build shared libraries" OFF)

include(cmake/ProjectFiles)

# $<INSTALL_INTERFACE:...> is exported using install(EXPORT)
# $<BUILD_INTERFACE:...> is exported using export(), or when the target is used by another target in the same buildsystem
macro(add_relative_include_dir TARGET VISIBILITY_BUILD VISIBILITY_INSTALL NEW_ELEM)
  target_include_directories(${TARGET}
    ${VISIBILITY_BUILD} "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/${NEW_ELEM}>"
    ${VISIBILITY_INSTALL} "$<INSTALL_INTERFACE:$<INSTALL_PREFIX>/${CMAKE_INSTALL_INCLUDEDIR}/${NEW_ELEM}>"
  )
endmacro(add_relative_include_dir)

add_library(${STATIC_LIB_NAME} STATIC
  ${compact_enc_det_SOURCES}
)

# path to third_party/ced/src/compact_enc_det/compact_enc_det.h
add_relative_include_dir(${STATIC_LIB_NAME} PUBLIC PUBLIC "chromium")

add_relative_include_dir(${STATIC_LIB_NAME} PUBLIC PUBLIC "chromium/third_party/ced/src")

target_include_directories(${STATIC_LIB_NAME} PRIVATE
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>"
)

set_property(TARGET ${STATIC_LIB_NAME} PROPERTY CXX_STANDARD 17)

# POSITION_INDEPENDENT_CODE for -fPIC
# -fPIC may be used to link static library into dynamic one
# Position-independent code requires special support,
# and therefore works only on certain machines.
# see https://akkadia.org/drepper/dsohowto.pdf
set_property(TARGET ${STATIC_LIB_NAME} PROPERTY POSITION_INDEPENDENT_CODE ON)

if(ENABLE_MSAN OR ENABLE_TSAN OR ENABLE_ASAN OR ENABLE_UBSAN)
  message(STATUS
    "Using sanitizers")

  if(cmake_build_type_tolower MATCHES "release" )
    message(WARNING
      "Disable sanitizers in Release builds")
  endif()

  # use llvm_tools from conan
  find_program_helper(llvm-symbolizer
    PATHS
      #${CONAN_BIN_DIRS}
      ${CONAN_BIN_DIRS_LLVM_TOOLS}
    NO_SYSTEM_ENVIRONMENT_PATH
    NO_CMAKE_SYSTEM_PATH
    ${ARGUMENTS_UNPARSED_ARGUMENTS}
    REQUIRED
    OUT_VAR LLVM_SYMBOLIZER_PROGRAM
    VERBOSE TRUE
  )
  check_sanitizer_options(
    ENABLE_TSAN ${ENABLE_TSAN}
    ENABLE_ASAN ${ENABLE_ASAN}
    ENABLE_MSAN ${ENABLE_MSAN}
    ENABLE_UBSAN ${ENABLE_UBSAN}
    LLVM_SYMBOLIZER_PROGRAM ${LLVM_SYMBOLIZER_PROGRAM}
  )
else()
  message(STATUS
    "Disabled sanitizers")
endif()

if(ENABLE_MSAN)
  message(STATUS "enabling MSAN on ${STATIC_LIB_NAME}")
  add_msan_static_link(${STATIC_LIB_NAME})
  add_msan_definitions(${STATIC_LIB_NAME})
  add_msan_flags()
else(ENABLE_MSAN)
  message(STATUS
    "Disabled ENABLE_MSAN")
endif(ENABLE_MSAN)

if(ENABLE_TSAN)
  message(STATUS "enabling TSAN on ${STATIC_LIB_NAME}")
  add_tsan_static_link(${STATIC_LIB_NAME})
  add_tsan_flags(${STATIC_LIB_NAME})
  add_tsan_definitions(${STATIC_LIB_NAME})
  add_tsan_flags()
else(ENABLE_TSAN)
  message(STATUS
    "Disabled ENABLE_TSAN")
endif(ENABLE_TSAN)

if(ENABLE_ASAN)
  message(STATUS "enabling ASAN on ${STATIC_LIB_NAME}")
  add_asan_static_link(${STATIC_LIB_NAME})
  add_asan_definitions(${STATIC_LIB_NAME})
  add_asan_flags()
else(ENABLE_ASAN)
  message(STATUS
    "Disabled ENABLE_ASAN")
endif(ENABLE_ASAN)

if(ENABLE_UBSAN)
  message(STATUS "enabling UBSAN on ${STATIC_LIB_NAME}")
  add_ubsan_static_link(${STATIC_LIB_NAME})
  add_ubsan_definitions(${STATIC_LIB_NAME})
  add_ubsan_flags()
else(ENABLE_UBSAN)
  message(STATUS
    "Disabled ENABLE_UBSAN")
endif(ENABLE_UBSAN)

if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
  set(CED_CMAKE_CXX_FLAGS "${CED_CMAKE_CXX_FLAGS} -std=c++11 -Wno-narrowing")
elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  set(CED_CMAKE_CXX_FLAGS "${CED_CMAKE_CXX_FLAGS} -std=c++11 -Wno-c++11-narrowing")
elseif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
  if (NOT BUILD_SHARED_LIBS)
    foreach(flag_var
            CED_CMAKE_CXX_FLAGS CED_CMAKE_CXX_FLAGS_DEBUG CED_CMAKE_CXX_FLAGS_RELEASE
            CED_CMAKE_CXX_FLAGS_MINSIZEREL CED_CMAKE_CXX_FLAGS_RELWITHDEBINFO)
      if(${flag_var} MATCHES "/MD")
        string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
      endif(${flag_var} MATCHES "/MD")
    endforeach(flag_var)
  endif()
else()
  set(CED_CMAKE_CXX_FLAGS "${CED_CMAKE_CXX_FLAGS} -std=c++11 -Wno-c++11-narrowing")
endif()

# https://stackoverflow.com/a/28294859/10904212
message(STATUS "CED_CMAKE_CXX_FLAGS=${CED_CMAKE_CXX_FLAGS}")
if(CED_CMAKE_CXX_FLAGS)
  string(REPLACE " " ";" CED_REPLACED_FLAGS ${CED_CMAKE_CXX_FLAGS})
  target_compile_options(${STATIC_LIB_NAME} PRIVATE
    ${CED_REPLACED_FLAGS}
  )
endif(CED_CMAKE_CXX_FLAGS)

if(WIN32)
  target_compile_definitions(${STATIC_LIB_NAME} PRIVATE
    UNICODE
    _UNICODE
    STRICT
    NOMINMAX)
  set(CED_THREADING threadwin)
elseif(TARGET_LINUX)
  set(CED_THREADING thread)
  list(APPEND EXTRA_TARGET_LINK_LIBRARIES -pthread)
elseif(TARGET_EMSCRIPTEN)
  # skip
else()
  message(FATAL_ERROR "platform not supported")
endif()

if(TARGET_EMSCRIPTEN)
  # use PROPERTY CXX_STANDARD 17
else()
  target_compile_features(${STATIC_LIB_NAME}
    PUBLIC cxx_auto_type
    PRIVATE cxx_variadic_templates)
endif()

set(DEBUG_LIBRARY_SUFFIX "-d")
set_target_properties(${STATIC_LIB_NAME}
  PROPERTIES
    ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}" # TODO: /lib
    LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}" # TODO: /lib
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}" # TODO: /bin
    OUTPUT_NAME "${STATIC_LIB_NAME}$<$<CONFIG:Debug>:${DEBUG_LIBRARY_SUFFIX}>"
)

# install and export steps
include(cmake/Exports)
