CMAKE_MINIMUM_REQUIRED(VERSION 3.10)

PROJECT("SmartCGMS")

message("========================")
message("Building dependencies...")
message("========================")

# The dependencies below are configured by nested CMake runs, which do not inherit our cache.
# ExcelFormat and xlnt still declare cmake_minimum_required(VERSION <3.5); CMake 4 refuses those
# outright, so the policy floor has to be handed down explicitly when the caller sets one.
set(DEPENDENCY_CMAKE_ARGS
        -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
        -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
        -DCMAKE_PREFIX_PATH=${CMAKE_PREFIX_PATH}
        -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
)
if(DEFINED CMAKE_POLICY_VERSION_MINIMUM)
    list(APPEND DEPENDENCY_CMAKE_ARGS -DCMAKE_POLICY_VERSION_MINIMUM=${CMAKE_POLICY_VERSION_MINIMUM})
endif()
if(APPLE AND DEFINED CMAKE_OSX_DEPLOYMENT_TARGET)
    list(APPEND DEPENDENCY_CMAKE_ARGS -DCMAKE_OSX_DEPLOYMENT_TARGET=${CMAKE_OSX_DEPLOYMENT_TARGET})
endif()
# GCC 13+ stopped providing <cstdint> transitively. The vendored xlnt uses std::uint8_t without
# including it (include/xlnt/styles/color.hpp), which libc++ still supplies indirectly - hence this
# breaks only on Linux. Force-include it for the dependency builds instead of patching a submodule.
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    list(APPEND DEPENDENCY_CMAKE_ARGS "-DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS} -include cstdint")
endif()

# A failing nested CMake run only surfaces much later, as a confusing "missing and no known rule
# to make it" from the generator, so stop as soon as one of the steps below reports an error.
# On failure execute_process yields either an exit code or an error string, hence the STREQUAL.
function(CHECK_DEPENDENCY_STEP STEP_RESULT)
    if(NOT STEP_RESULT STREQUAL "0")
        message(FATAL_ERROR "Building the dependencies failed (see the output above); the step reported '${STEP_RESULT}'.")
    endif()
endfunction()

#########################################################################################
# ExcelFormat

execute_process(
        COMMAND cmake -S . -B ./build/ -G ${CMAKE_GENERATOR}
        ${DEPENDENCY_CMAKE_ARGS}
        -DCMAKE_DEBUG_POSTFIX=d # Without this debug build is unable to find the lib file
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/deps/ExcelFormat
        RESULT_VARIABLE DEPENDENCY_STEP_RESULT
)
CHECK_DEPENDENCY_STEP("${DEPENDENCY_STEP_RESULT}")

message(STATUS "> ExcelFormat has been configured.")


execute_process(
        COMMAND cmake --build ./build/ --parallel
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/deps/ExcelFormat
        RESULT_VARIABLE DEPENDENCY_STEP_RESULT
)
CHECK_DEPENDENCY_STEP("${DEPENDENCY_STEP_RESULT}")

message(STATUS "> ExcelFormat has been built.")

#########################################################################################
# XLNT
file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/compiled)

execute_process(
        COMMAND cmake -S . -B ./build/ -G ${CMAKE_GENERATOR}
        ${DEPENDENCY_CMAKE_ARGS}

        # Make sure xlnt's output is place into the "compiler" directory (the xlnt.dll)
        -DCMAKE_RUNTIME_OUTPUT_DIRECTORY=${CMAKE_CURRENT_BINARY_DIR}/compiled
        -DCMAKE_LIBRARY_OUTPUT_DIRECTORY=${CMAKE_CURRENT_BINARY_DIR}/compiled
        -DCMAKE_ARCHIVE_OUTPUT_DIRECTORY=${CMAKE_CURRENT_BINARY_DIR}/compiled

        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/deps/xlnt
        RESULT_VARIABLE DEPENDENCY_STEP_RESULT
)
CHECK_DEPENDENCY_STEP("${DEPENDENCY_STEP_RESULT}")

message(STATUS "> xlnt has been configured.")

execute_process(
        COMMAND cmake --build ./build/ --parallel
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/deps/xlnt
        RESULT_VARIABLE DEPENDENCY_STEP_RESULT
)
CHECK_DEPENDENCY_STEP("${DEPENDENCY_STEP_RESULT}")

message(STATUS "> xlnt has been built.")

#########################################################################################
# EIGEN 3

execute_process(
        COMMAND cmake -S . -B ./build/ -G ${CMAKE_GENERATOR}
        ${DEPENDENCY_CMAKE_ARGS}
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/deps/Eigen3
        RESULT_VARIABLE DEPENDENCY_STEP_RESULT
)
CHECK_DEPENDENCY_STEP("${DEPENDENCY_STEP_RESULT}")

message(STATUS "> Eigen3 has been configured.")

message("======================")
message("Configuring Project...")
message("======================")

# Setting this to OFF because of the "distributed4scgms" sub-project, which already contains a copy of Pagmo
set(PAGMO4SCGMS_USE_INTERNAL_PAGMO OFF)

ADD_SUBDIRECTORY("common")

ADD_SUBDIRECTORY("core")

IF (NOT DEFINED BUILD_CORE_ONLY OR NOT BUILD_CORE_ONLY)
    ADD_SUBDIRECTORY("console")
    ADD_SUBDIRECTORY("desktop")
ENDIF ()

ADD_SUBDIRECTORY("wrappers")

ADD_SUBDIRECTORY("PathFinder_test")

