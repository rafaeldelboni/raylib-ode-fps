# Setup
cmake_minimum_required(VERSION 3.17)
project(RaylibOdeFps)
include(CTest)
include(FetchContent)
set(FETCHCONTENT_QUIET 0)

# Download / Include Unity Test Runner
FetchContent_Declare(
  unity
  GIT_REPOSITORY https://github.com/ThrowTheSwitch/Unity.git
  GIT_SHALLOW    TRUE
  GIT_PROGRESS   TRUE
  GIT_TAG        master)

FetchContent_Declare(
  raylib
  GIT_REPOSITORY https://github.com/raysan5/raylib.git
  GIT_SHALLOW    TRUE
  GIT_PROGRESS   TRUE
  GIT_TAG        master)
set(BUILD_EXAMPLES OFF CACHE BOOL "" FORCE) # don't build the supplied examples

FetchContent_Declare(
  ode
  GIT_REPOSITORY https://bitbucket.org/odedevs/ode.git
  GIT_SHALLOW    TRUE
  GIT_PROGRESS   TRUE
  GIT_TAG        master)
set(ODE_WITH_DEMOS OFF CACHE BOOL "" FORCE)
set(ODE_WITH_TESTS OFF CACHE BOOL "" FORCE)
set(ODE_DOUBLE_PRECISION OFF CACHE BOOL "" FORCE)
string(APPEND CMAKE_CXX_FLAGS " -fPIC")

FetchContent_MakeAvailable(unity raylib ode)

# Output directory structure configuration.
set(EXECUTABLE_OUTPUT_PATH "${PROJECT_BINARY_DIR}/bin")
set(LIBRARY_OUTPUT_PATH "${PROJECT_BINARY_DIR}/bin")
set(TEST_OUTPUT_PATH "${EXECUTABLE_OUTPUT_PATH}/test")

# Add symbolic link to resources folder in build
add_custom_target(resources)
FILE(TO_NATIVE_PATH "${PROJECT_SOURCE_DIR}/resources" source)
FILE(TO_NATIVE_PATH "${PROJECT_BINARY_DIR}/resources" destination)
IF (WIN32)
  add_custom_command(
    TARGET resources POST_BUILD
    COMMAND mklink /D ${destination} ${source}
    DEPENDS ${destination}
    COMMENT "symbolic link resources folder from ${source} => ${destination}"
    )
ELSE()
  add_custom_command(
    TARGET resources POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E create_symlink ${source} ${destination}
    DEPENDS ${destination}
    COMMENT "symbolic link resources folder from ${source} => ${destination}"
    )
ENDIF()

# Generates a compile_commands.json file containing the exact compiler calls
IF (NOT WIN32)
  set(CMAKE_EXPORT_COMPILE_COMMANDS 1)
  FILE(TO_NATIVE_PATH "${PROJECT_BINARY_DIR}/compile_commands.json" sourceJson)
  FILE(TO_NATIVE_PATH "${PROJECT_SOURCE_DIR}/build/compile_commands.json" destinationJson)

  add_custom_command(TARGET resources POST_BUILD
    COMMAND "${CMAKE_COMMAND}" -E copy 
    ${sourceJson} ${destinationJson}
    COMMENT "Copying compile_commands.json to build directory")
ENDIF()

# Project-wide include directory configuration.
include_directories("${PROJECT_SOURCE_DIR}/src")

# Project library source lives in src/.
add_subdirectory(src)

# Project apps that may link to project libraries live in app/.
add_subdirectory(app)

# CTest sets BUILD_TESTING option to ON by default.
# Test-related configuration goes here.
if (BUILD_TESTING)
  enable_testing()
  add_subdirectory(test)

  # COVERAGE
  if (${CMAKE_BUILD_TYPE} MATCHES "Coverage")
    if ("${CMAKE_C_COMPILER_ID}" STREQUAL "GNU")
      add_compile_options(--coverage)
      link_libraries(gcov)
    else ()
      message(FATAL_ERROR "Only GCC is supported for coverage")
    endif ()
  endif ()

  add_custom_target(coverage)
  add_custom_command(
    TARGET coverage
    COMMAND make test
    COMMAND lcov --capture --directory "${PROJECT_BINARY_DIR}/src/CMakeFiles"
    --output-file "${PROJECT_BINARY_DIR}/coverage.info" --rc lcov_branch_coverage=1 --rc lcov_excl_br_line='assert'
    COMMAND lcov --remove "${PROJECT_BINARY_DIR}/coverage.info" '/usr/*' '*test*'
    --output-file "${PROJECT_BINARY_DIR}/coverage.info" --rc lcov_branch_coverage=1 --rc lcov_excl_br_line='assert'
    COMMAND lcov --list "${PROJECT_BINARY_DIR}/coverage.info" --rc lcov_branch_coverage=1 --rc lcov_excl_br_line='assert'
    )
endif()
