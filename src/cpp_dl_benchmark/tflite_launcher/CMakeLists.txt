find_path(TENSORFLOW_SRC_DIR "tensorflow/lite/interpreter.h" CMAKE_FIND_ROOT_PATH_BOTH)

# If the file in a directory is found the result is stored in the variable and the search will not be repeated unless the variable is cleared.
# If nothing is found, the result will be <VAR>-NOTFOUND.
unset(TFLITE_LIB_FOUND CACHE)

#default name for tflite lib built with cmake
set(TFLITE_LIB_NAME "libtensorflow-lite.so")
find_path(TFLITE_LIB_FOUND ${TFLITE_LIB_NAME} HINT ${TFLITE_BUILD_DIR} CMAKE_FIND_ROOT_PATH_BOTH)
if(TFLITE_LIB_FOUND STREQUAL TFLITE_LIB_FOUND-NOTFOUND)
    # bazel build has another name for tflite lib
    set(TFLITE_LIB_NAME "libtensorflowlite.so")
    find_path(TFLITE_LIB_FOUND ${TFLITE_LIB_NAME} HINT ${TFLITE_BUILD_DIR} CMAKE_FIND_ROOT_PATH_BOTH)
    if(TFLITE_LIB_FOUND STREQUAL TFLITE_LIB_FOUND-NOTFOUND)
        message(SEND_ERROR "Couldn't find libtensorflowlite.so or libtensorflow-lite.so")
    endif()
endif()


if (TENSORFLOW_SRC_DIR AND TFLITE_BUILD_DIR)
    # find libraries generated by bazel build
    if(TFLITE_LIB_NAME STREQUAL "libtensorflowlite.so")
        get_filename_component(BAZEL_BUILD_ROOT "${TFLITE_BUILD_DIR}/../.." ABSOLUTE)
        set(FLATBUFFERS_INCLUDE_DIR
            "${BAZEL_BUILD_ROOT}/external/flatbuffers/src/_virtual_includes/flatbuffers")
        find_library(TFLITE_C_LIBRARY libtensorflowlite_c.so
                    ${TFLITE_BUILD_DIR}/c
                    CMAKE_FIND_ROOT_PATH_BOTH)
        find_library(TFLITE_GPU_LIBRARY libtensorflowlite_gpu_delegate.so
                     ${TFLITE_BUILD_DIR}/delegates/gpu
                     CMAKE_FIND_ROOT_PATH_BOTH)
        find_library(TFLITE_FLEX_LIBRARY libtensorflowlite_flex.so
                    ${TFLITE_BUILD_DIR}/delegates/flex
                    CMAKE_FIND_ROOT_PATH_BOTH)

        if(TFLITE_GPU_LIBRARY STREQUAL TFLITE_GPU_LIBRARY-NOTFOUND)
            unset(TFLITE_GPU_LIBRARY CACHE)
            message(WARNING "libtensorflowlite_gpu_delegate.so wasn't found, build without GPU support.")
        endif()

        if(TFLITE_FLEX_LIBRARY STREQUAL TFLITE_FLEX_LIBRARY-NOTFOUND)
            unset(TFLITE_FLEX_LIBRARY CACHE)
            message(WARNING "libtensorflowlite_flex.so wasn't found, build without TensorFlow operators support.")
        endif()
    else()
        set(FLATBUFFERS_INCLUDE_DIR "${TFLITE_BUILD_DIR}/flatbuffers/include")
        set(ABSEIL_DIR "${TFLITE_BUILD_DIR}/abseil-cpp")

        set(TSL_DIR ${TENSORFLOW_SRC_DIR}/tensorflow)
        # starting from tflite 2.15 tsl moved into thirdparty
        if(NOT EXISTS ${TSL_DIR}/tsl/util/stats_calculator.cc)
            set(TSL_DIR ${TENSORFLOW_SRC_DIR}/third_party/xla/third_party/tsl)
        endif()
    endif()

    set(TFLITE_INCLUDE_DIR "${TENSORFLOW_SRC_DIR};${FLATBUFFERS_INCLUDE_DIR};${ABSEIL_DIR};${TSL_DIR}")

    find_library(TFLITE_LIB_PATH "${TFLITE_LIB_NAME}"
                 ${TFLITE_BUILD_DIR}
                 CMAKE_FIND_ROOT_PATH_BOTH)

    set(TFLITE_LIBRARY "tensorflow-lite")
    add_library(${TFLITE_LIBRARY} SHARED IMPORTED)
    set_target_properties(${TFLITE_LIBRARY} PROPERTIES
                          INTERFACE_INCLUDE_DIRECTORIES "${TFLITE_INCLUDE_DIR}"
                          IMPORTED_LOCATION ${TFLITE_LIB_PATH}
                          IMPORTED_IMPLIB ${TFLITE_LIB_PATH})

    set(TFLITE_SOURCE_DIR ${TENSORFLOW_SRC_DIR}/tensorflow/lite)

    set(PROFILING_SOURCES
        ${TSL_DIR}/tsl/util/stats_calculator.cc
        ${TFLITE_SOURCE_DIR}/profiling/memory_info.cc
        ${TFLITE_SOURCE_DIR}/profiling/memory_usage_monitor.cc
        ${TFLITE_SOURCE_DIR}/profiling/profile_buffer.h
        ${TFLITE_SOURCE_DIR}/profiling/profile_summarizer.cc
        ${TFLITE_SOURCE_DIR}/profiling/profile_summary_formatter.cc
        ${TFLITE_SOURCE_DIR}/profiling/root_profiler.cc
        ${TFLITE_SOURCE_DIR}/profiling/telemetry/profiler.cc
        ${TFLITE_SOURCE_DIR}/profiling/telemetry/telemetry.cc
        ${TFLITE_SOURCE_DIR}/profiling/time.cc
    )
    if(EXISTS ${TFLITE_SOURCE_DIR}/profiling/profile_buffer.cc)
        list(APPEND PROFILING_SOURCES ${TFLITE_SOURCE_DIR}/profiling/profile_buffer.cc)
    endif()

    file(GLOB_RECURSE DEPS "${TFLITE_BUILD_DIR}/*.so")
    list(REMOVE_ITEM DEPS ${TFLITE_LIB_PATH})
    foreach(lib IN LISTS DEPS)
        target_link_libraries(${TFLITE_LIBRARY} INTERFACE ${lib})
    endforeach()

endif()

add_subdirectory(custom_ops)

file(GLOB_RECURSE SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/*.cpp)
file(GLOB_RECURSE HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/*.hpp)

list(APPEND SOURCES ${PROFILING_SOURCES})

if (BUILD_TFLITE_LAUNCHER)
    add_launcher(NAME tflite_benchmark
                 SOURCES ${SOURCES}
                 HEADERS ${HEADERS}
                 INCLUDE_DIRECTORIES "${CMAKE_CURRENT_SOURCE_DIR}/include"
                 DEPENDENCIES
                    -Wl,-rpath-link,${TFLITE_BUILD_DIR} # workaround for custom RISC-V toolchains
                    ${TFLITE_LIBRARY} ${TFLITE_C_LIBRARY}
                    -Wl,--no-as-needed # Need --no-as-needed to link tensorflowlite_flex
                    ${TFLITE_FLEX_LIBRARY}  tflite_custom_ops
                 DEFINITIONS TFLITE_WITH_DEFAULT_BACKEND)
endif()

if (BUILD_TFLITE_XNNPACK_LAUNCHER)
    add_launcher(NAME tflite_xnnpack_benchmark
                 SOURCES ${SOURCES}
                 HEADERS ${HEADERS}
                 INCLUDE_DIRECTORIES "${CMAKE_CURRENT_SOURCE_DIR}/include"
                 DEPENDENCIES
                    -Wl,-rpath-link,${TFLITE_BUILD_DIR} # workaround for custom RISC-V toolchains
                    ${TFLITE_LIBRARY} ${TFLITE_C_LIBRARY}
                    -Wl,--no-as-needed # Need --no-as-needed to link tensorflowlite_flex
                    ${TFLITE_FLEX_LIBRARY} tflite_custom_ops
                 DEFINITIONS TFLITE_WITH_XNNPACK_BACKEND)
endif()

if (BUILD_TFLITE_GPU_LAUNCHER)
    add_launcher(NAME tflite_gpu_delegate_benchmark
                 SOURCES ${SOURCES}
                 HEADERS ${HEADERS}
                 INCLUDE_DIRECTORIES "${CMAKE_CURRENT_SOURCE_DIR}/include"
                 DEPENDENCIES
                    -Wl,-rpath-link,${TFLITE_BUILD_DIR} # workaround for custom RISC-V toolchains
                    ${TFLITE_LIBRARY} ${TFLITE_C_LIBRARY}
                    -Wl,--no-as-needed # Need --no-as-needed to link tensorflowlite_flex
                    ${TFLITE_FLEX_LIBRARY} ${TFLITE_GPU_LIBRARY} tflite_custom_ops
                 DEFINITIONS TFLITE_WITH_GPU_DELEGATE)
endif()
