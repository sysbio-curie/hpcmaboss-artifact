cmake_minimum_required(VERSION 3.18)

project(MaBoSS.GPU VERSION 0.1 LANGUAGES CUDA CXX C)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_CUDA_STANDARD 17)
set(CMAKE_CUDA_STANDARD_REQUIRED ON)

# Options
if (NOT MAX_NODES)
    set(MAX_NODES 512)
endif()

add_compile_definitions(MAX_NODES=${MAX_NODES})

# Find CC of installed GPUs
include(FindCUDA/select_compute_arch)
CUDA_DETECT_INSTALLED_GPUS(INSTALLED_GPU_CCS_1)
string(STRIP "${INSTALLED_GPU_CCS_1}" INSTALLED_GPU_CCS_2)
string(REPLACE " " ";" INSTALLED_GPU_CCS_3 "${INSTALLED_GPU_CCS_2}")
string(REPLACE "." "" CUDA_ARCH_LIST "${INSTALLED_GPU_CCS_3}")
SET(CMAKE_CUDA_ARCHITECTURES ${CUDA_ARCH_LIST})

### Target MaBoSSGCore ###

set(PARSER_GENERATED_PATH ${CMAKE_CURRENT_SOURCE_DIR}/src/parser/generated)
set(JIT_INCLUDE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/src/jit_kernels/include)
set(JIT_FATBIN_PATH ${CMAKE_CURRENT_SOURCE_DIR}/src/jit_kernels/fatbin)

add_custom_target(make-parser-gen-dir ALL
    COMMAND ${CMAKE_COMMAND} -E make_directory ${PARSER_GENERATED_PATH})

add_custom_target(make-jit-inc-dir ALL
    COMMAND ${CMAKE_COMMAND} -E make_directory ${JIT_INCLUDE_PATH})

add_custom_target(make-jit-fat-dir ALL
    COMMAND ${CMAKE_COMMAND} -E make_directory ${JIT_FATBIN_PATH})

find_package(FLEX REQUIRED)
FLEX_TARGET(maboss_parser src/parser/scanner.ll ${PARSER_GENERATED_PATH}/scanner.cpp
            DEFINES_FILE ${PARSER_GENERATED_PATH}/scanner.h)

find_package(BISON REQUIRED)
BISON_TARGET(maboss_parser src/parser/parser.yy ${PARSER_GENERATED_PATH}/parser.cpp
             DEFINES_FILE ${PARSER_GENERATED_PATH}/parser.h)

set(JIT_INCLUDE_FILES ${CMAKE_CURRENT_SOURCE_DIR}/src/trajectory_status.h 
                      ${CMAKE_CURRENT_SOURCE_DIR}/src/state_word.h)

set(JIT_FATBIN_FILES ${CMAKE_CURRENT_SOURCE_DIR}/src/jit_kernels/simulation.cu
                     ${CMAKE_CURRENT_SOURCE_DIR}/src/jit_kernels/final_states.cu
                     ${CMAKE_CURRENT_SOURCE_DIR}/src/jit_kernels/window_average_small.cu)

foreach(path ${JIT_INCLUDE_FILES})
    get_filename_component(name ${path} NAME)

    add_custom_command(
        OUTPUT ${JIT_INCLUDE_PATH}/${name}
        COMMAND ${CMAKE_COMMAND} -Dinput_file=${path} -Doutput_file=${JIT_INCLUDE_PATH}/${name} -P ${CMAKE_CURRENT_SOURCE_DIR}/cmake/stringify.cmake
        MAIN_DEPENDENCY ${path}
    )

    list(APPEND JIT_RESULT_FILES ${JIT_INCLUDE_PATH}/${name})
endforeach()

add_executable(dumpbin ${CMAKE_CURRENT_SOURCE_DIR}/cmake/dumpbin.c)

foreach(path ${JIT_FATBIN_FILES})
    get_filename_component(name ${path} NAME_WE)

    add_custom_command(
        OUTPUT ${JIT_FATBIN_PATH}/${name}.fatbin 
        COMMAND ${CMAKE_CUDA_COMPILER} -arch lto_${CUDA_ARCH_LIST} -rdc=true -fatbin
            -o ${JIT_FATBIN_PATH}/${name}.fatbin 
            ${path}
        MAIN_DEPENDENCY ${path}
    )

    add_custom_command(
        OUTPUT ${JIT_INCLUDE_PATH}/${name}.fatbin.h
        COMMAND dumpbin ${JIT_FATBIN_PATH}/${name}.fatbin ${JIT_INCLUDE_PATH}/${name}.fatbin.h
        MAIN_DEPENDENCY ${JIT_FATBIN_PATH}/${name}.fatbin
        DEPENDS dumpbin
    )

    list(APPEND JIT_RESULT_FILES ${JIT_INCLUDE_PATH}/${name}.fatbin.h)
endforeach()

add_custom_target(jit_generated DEPENDS ${JIT_RESULT_FILES})

add_compile_definitions(CUDA_INC_DIR="${CMAKE_CUDA_TOOLKIT_INCLUDE_DIRECTORIES}")
add_compile_definitions(CUDA_CC="${CUDA_ARCH_LIST}")

# Thrust has a linker issue when device_ptr being compiled using gcc in one unit and using nvcc in another
# A function with device_ptr parameter is defined in stats_composite.cpp and required from simulation_runner.cu -> error
set_source_files_properties("src/statistics/stats_composite.cpp" PROPERTIES LANGUAGE CUDA)

file(GLOB_RECURSE src_files "src/*")
list(FILTER src_files EXCLUDE REGEX ".*main\\..*")
list(FILTER src_files EXCLUDE REGEX ".*jit_kernels.*")
add_library(MaBoSSGCore ${src_files} ${FLEX_maboss_parser_OUTPUTS} ${BISON_maboss_parser_OUTPUTS})
target_include_directories(MaBoSSGCore PUBLIC ${CMAKE_CUDA_TOOLKIT_INCLUDE_DIRECTORIES} ${jitify_SOURCE_DIR})
target_link_libraries(MaBoSSGCore PUBLIC cuda cudart nvrtc nvJitLink)
add_dependencies(MaBoSSGCore jit_generated)

### Target MaBoSSGCore ###


### Target unit_MaBoSS.GPU ###

include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/609281088cfefc76f9d0ce82e1ff6c30cc3591e5.zip
)
# For Windows: Prevent overriding the parent project's compiler/linker settings
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

file(GLOB_RECURSE test_files "test/*")
add_executable(unit_MaBoSS.GPU ${test_files})

target_link_libraries(unit_MaBoSS.GPU PRIVATE
	gtest_main gmock_main MaBoSSGCore)

target_include_directories(unit_MaBoSS.GPU PUBLIC
	"src")

add_custom_command(TARGET unit_MaBoSS.GPU POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
    ${CMAKE_CURRENT_SOURCE_DIR}/data $<TARGET_FILE_DIR:${PROJECT_NAME}>/data)

enable_testing()
include(GoogleTest)
gtest_discover_tests(unit_MaBoSS.GPU)

### Target unit_MaBoSS.GPU ###


### Target MaBoSS.GPU ###

add_executable(MaBoSS.GPU src/main.cpp)
target_link_libraries(MaBoSS.GPU MaBoSSGCore)

### Target MaBoSS.GPU ###

foreach(target MaBoSS.GPU unit_MaBoSS.GPU MaBoSSGCore)
	if(MSVC)
		target_compile_options(${target} PRIVATE $<$<COMPILE_LANGUAGE:CXX>:/W4 /bigobj>)
	else()
		target_compile_options(${target} PRIVATE $<$<COMPILE_LANGUAGE:CXX>:-Wall -Wextra -pedantic>)
	endif()
    target_compile_options(${target} PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:--extended-lambda --expt-relaxed-constexpr>)
#   target_compile_options(${target} PRIVATE $<$<COMPILE_LANGUAGE:CUDA>:--source-in-ptx -lineinfo>)
endforeach()
