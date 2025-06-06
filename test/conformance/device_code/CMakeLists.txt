# Copyright (C) 2023 Intel Corporation
# Part of the Unified-Runtime Project, under the Apache License v2.0 with LLVM Exceptions.
# See LICENSE.TXT
# SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception

if(NOT "${AMD_ARCH}" STREQUAL "" AND "${UR_CONFORMANCE_AMD_ARCH}" STREQUAL "")
    message(WARNING "Passing the HIP target architecture with AMD_ARCH is deprecated. "
                    "Please use UR_CONFORMANCE_AMD_ARCH instead.")
else()
    set(AMD_ARCH "${UR_CONFORMANCE_AMD_ARCH}")
endif()

if (WIN32)
  set(NULDEV NUL)
else()
  set(NULDEV /dev/null)
endif()

if("${AMD_ARCH}" STREQUAL "" AND "${TARGET_TRIPLES}" MATCHES "amd")
    find_package(RocmAgentEnumerator)
    if(NOT ROCM_AGENT_ENUMERATOR_FOUND)
        message(FATAL_ERROR
                "rocm_agent_enumerator could not be found so detecting target "
                "HIP device has failed. Set target device with UR_CONFORMANCE_AMD_ARCH "
                "or ensure rocm_agent_enumerator is available on the PATH.")
    endif()
    execute_process(COMMAND ${ROCM_AGENT_ENUMERATOR} OUTPUT_VARIABLE ROCM_AGENTS)
    string(REGEX MATCHALL "[A-Za-z0-9]+" ROCM_AGENT_LIST "${ROCM_AGENTS}")
    # rocm_agent_enumerator will return gfx000 to represent non-amd-gpu devices,
    # we should skip over these
    list(GET ROCM_AGENT_LIST 0 FIRST_ROCM_AGENT)
    if("${FIRST_ROCM_AGENT}" STREQUAL "gfx000")
        list(POP_FRONT ROCM_AGENT_LIST)
    endif()
    list(LENGTH ROCM_AGENT_LIST NUM_ROCM_AGENTS)
    if(${NUM_ROCM_AGENTS} EQUAL 0)
        message(FATAL_ERROR
                "No target HIP devices detected with rocm_agent_enumerator, "
                "specify a target with the UR_CONFORMANCE_AMD_ARCH variable.")
    elseif(${NUM_ROCM_AGENTS} GREATER 1)
        message(FATAL_ERROR
                "Multiple possible target HIP devices found: ${ROCM_AGENT_LIST} "
                "please specify which target to use by setting UR_CONFORMANCE_AMD_ARCH.")
    endif()
    list(GET ROCM_AGENT_LIST 0 AMD_ARCH)
endif()

macro(add_device_binary SOURCE_FILE)
    get_filename_component(KERNEL_NAME ${SOURCE_FILE} NAME_WE)
    set(DEVICE_BINARY_DIR "${UR_CONFORMANCE_DEVICE_BINARIES_DIR}/${KERNEL_NAME}")
    file(MAKE_DIRECTORY ${DEVICE_BINARY_DIR})
    if(UR_SYCL_LIBRARY_DIR)
        if(CMAKE_SYSTEM_NAME STREQUAL Linux)
            set(EXTRA_ENV LD_LIBRARY_PATH=${UR_SYCL_LIBRARY_DIR})
        elseif(CMAKE_SYSTEM_NAME STREQUAL Windows)
            set(EXTRA_ENV PATH=${UR_SYCL_LIBRARY_DIR};$ENV{PATH})
        else()
            set(EXTRA_ENV DYLD_FALLBACK_LIBRARY_PATH=${UR_SYCL_LIBRARY_DIR})
        endif()
    endif()

    # Convert build flags to a regular CMake list, splitting by unquoted white
    # space as necessary.
    separate_arguments(DPCXX_BUILD_FLAGS_LIST NATIVE_COMMAND "${UR_DPCXX_BUILD_FLAGS}")

    foreach(TRIPLE ${TARGET_TRIPLES})
        set(EXE_PATH "${DEVICE_BINARY_DIR}/${KERNEL_NAME}_${TRIPLE}")
        set(BIN_PATH "${DEVICE_BINARY_DIR}/${TRIPLE}.bin.0")

        if(${TRIPLE} MATCHES "amd")
            set(AMD_TARGET_BACKEND -Xsycl-target-backend=${TRIPLE})
            set(AMD_OFFLOAD_ARCH  --offload-arch=${AMD_ARCH})
            set(AMD_NOGPULIB -nogpulib)
        else()
            set(AMD_TARGET_BACKEND)
            set(AMD_OFFLOAD_ARCH)
            set(AMD_NOGPULIB)
        endif()
        # images are not yet supported in sycl on AMD
        if(${TRIPLE} MATCHES "amd" AND ${KERNEL_NAME} MATCHES "image_copy")
            continue()
        endif()

        # This seems to fail to build the SYCL binary due to the invalid asm
        if(${TRIPLE} MATCHES "cuda" AND ${KERNEL_NAME} MATCHES "build_failure")
            continue()
        endif()
        if(${TRIPLE} MATCHES "amd" AND ${KERNEL_NAME} MATCHES "build_failure")
            continue()
        endif()

        # HIP doesn't seem to provide the symbol
        # `_ZTSZZ4mainENKUlRN4sycl3_V17handlerEE_clES2_E11FixedSgSize` which
        # causes a build failure here
        if(${TRIPLE} MATCHES "amd" AND ${KERNEL_NAME} MATCHES "subgroup")
            continue()
        endif()

        # cuda and hip seem to do linking at compile time (rather than runtime)
        if(${TRIPLE} MATCHES "nvptx" AND ${KERNEL_NAME} MATCHES "linker_error")
            continue()
        endif()
        if(${TRIPLE} MATCHES "amd" AND ${KERNEL_NAME} MATCHES "linker_error")
            continue()
        endif()

        add_custom_command(OUTPUT ${BIN_PATH}
            COMMAND LD_LIBRARY_PATH=${UR_SYCL_LIBRARY_DIR}:$ENV{LD_LIBRARY_PATH} 
            ${UR_FOUND_DPCXX} -fsycl -fsycl-targets=${TRIPLE}
            -fsycl-device-code-split=off ${AMD_TARGET_BACKEND}
            ${AMD_OFFLOAD_ARCH} ${AMD_NOGPULIB} ${DPCXX_BUILD_FLAGS_LIST}
            ${SOURCE_FILE} -o ${EXE_PATH}

            COMMAND ${CMAKE_COMMAND} -E env ${EXTRA_ENV} ${UR_FOUND_DEVICE_CODE_EXTRACTOR} -q --stem="${TRIPLE}.bin" ${EXE_PATH}

            WORKING_DIRECTORY "${DEVICE_BINARY_DIR}"
            DEPENDS ${SOURCE_FILE} ${UR_DPCXX_DEPS}
        )
        add_custom_target(generate_${KERNEL_NAME}_${TRIPLE} DEPENDS ${BIN_PATH})
        add_dependencies(generate_device_binaries generate_${KERNEL_NAME}_${TRIPLE})
    endforeach()

    set(IH_PATH "${DEVICE_BINARY_DIR}/${KERNEL_NAME}.ih")
    add_custom_command(OUTPUT "${IH_PATH}"
        COMMAND ${UR_FOUND_DPCXX} -fsycl -fsycl-device-code-split=off
        -fsycl-device-only -c -Xclang -fsycl-int-header="${IH_PATH}"
        ${DPCXX_BUILD_FLAGS_LIST} ${SOURCE_FILE} -o ${NULDEV}

        WORKING_DIRECTORY "${DEVICE_BINARY_DIR}"
        DEPENDS ${SOURCE_FILE}
    )
    list(APPEND DEVICE_IHS ${IH_PATH})
    list(APPEND DEVICE_CODE_SOURCES ${SOURCE_FILE})
endmacro()

add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/bar.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/device_global.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/fill.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/fill_2d.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/fill_3d.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/fill_usm.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/fill_usm_2d.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/foo.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/image_copy.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/inc.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/increment.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/mean.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/cpy_and_mult.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/cpy_and_mult_usm.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/multiply.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/spec_constant.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/spec_constant_multiple.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/usm_ll.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/saxpy.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/saxpy_usm.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/indexers_usm.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/build_failure.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/fixed_sg_size.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/fixed_wg_size.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/max_wg_size.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/sequence.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/standard_types.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/subgroup.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/linker_error.cpp)
add_device_binary(${CMAKE_CURRENT_SOURCE_DIR}/saxpy_usm_local_mem.cpp)

set(KERNEL_HEADER ${UR_CONFORMANCE_DEVICE_BINARIES_DIR}/kernel_entry_points.h)
add_custom_command(OUTPUT ${KERNEL_HEADER}
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/scripts
    COMMAND ${Python3_EXECUTABLE} generate_kernel_header.py
    -o ${KERNEL_HEADER} ${DEVICE_CODE_SOURCES}
    DEPENDS ${PROJECT_SOURCE_DIR}/scripts/generate_kernel_header.py
    ${DEVICE_CODE_SOURCES} ${DEVICE_IHS})
add_custom_target(kernel_names_header DEPENDS ${KERNEL_HEADER})
add_dependencies(generate_device_binaries kernel_names_header)
