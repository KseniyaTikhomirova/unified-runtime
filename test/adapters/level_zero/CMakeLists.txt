# Copyright (C) 2024 Intel Corporation
# Part of the Unified-Runtime Project, under the Apache License v2.0 with LLVM Exceptions.
# See LICENSE.TXT
# SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception

function(add_adapter_tests adapter)
    if(NOT UR_FOUND_DPCXX)
        # Tests that require kernels can't be used if we aren't generating
        # device binaries
        message(WARNING
            "UR_DPCXX is not defined, skipping some adapter tests for ${adapter}")
    else()
        add_adapter_test(${adapter}
            FIXTURE KERNELS
            SOURCES
                urProgramLink.cpp
                urKernelCreateWithNativeHandle.cpp
                urEventCreateWithNativeHandle.cpp
            ENVIRONMENT
                "UR_ADAPTERS_FORCE_LOAD=\"$<TARGET_FILE:ur_adapter_${adapter}>\""
        )
        # TODO: valgrind tests require very new environment.
        # Enable once all L0 runners are updated.
        # add_adapter_memcheck_test(${adapter}
        #    ENVIRONMENT
        #         "UR_ADAPTERS_FORCE_LOAD=\"$<TARGET_FILE:ur_adapter_${adapter}>\""
        # )

        target_link_libraries(test-adapter-${adapter} PRIVATE
            LevelZeroLoader
            LevelZeroLoader-Headers
            ComputeRuntimeLevelZero-Headers
        )

        target_include_directories(test-adapter-${adapter} PRIVATE
            ${PROJECT_SOURCE_DIR}/source
            ${PROJECT_SOURCE_DIR}/source/adapters/${adapter}
            LevelZeroLoader-Headers
        )

        add_dependencies(test-adapter-${adapter}
            generate_device_binaries kernel_names_header)


        if("${adapter}" STREQUAL "level_zero")
            add_adapter_test(level_zero_enqueue_alloc
                FIXTURE KERNELS
                SOURCES
                    enqueue_alloc.cpp
                ENVIRONMENT
                    "UR_ADAPTERS_FORCE_LOAD=\"$<TARGET_FILE:ur_adapter_level_zero>\""
            )
        endif()
    endif()

    if(NOT WIN32 AND NOT UR_STATIC_ADAPTER_L0)
        # TODO: adjust to work for v2
        if("${adapter}" STREQUAL "level_zero")
            add_adapter_test(${adapter}_ze_calls
                FIXTURE DEVICES
                SOURCES
                    event_cache_tests.cpp
                ENVIRONMENT
                    "UR_ADAPTERS_FORCE_LOAD=\"$<TARGET_FILE:ur_adapter_${adapter}>\""
                    "ZE_ENABLE_TRACING_LAYER=1"
            )

            target_link_libraries(test-adapter-${adapter}_ze_calls PRIVATE LevelZeroLoader LevelZeroLoader-Headers)
        endif()

        # TODO: enable for v2 once driver issue is fixed
        if("${adapter}" STREQUAL "level_zero")
            add_adapter_test(${adapter}_multi_queue
                FIXTURE DEVICES
                SOURCES
                    multi_device_event_cache_tests.cpp
                ENVIRONMENT
                    "UR_ADAPTERS_FORCE_LOAD=\"$<TARGET_FILE:ur_adapter_${adapter}>\""
                    "ZE_ENABLE_TRACING_LAYER=1"
            )

            target_link_libraries(test-adapter-${adapter}_multi_queue PRIVATE LevelZeroLoader LevelZeroLoader-Headers)
        endif()
    endif()

    # TODO: debug for v2
    if("${adapter}" STREQUAL "level_zero")
        add_adapter_test(${adapter}_ipc
            FIXTURE DEVICES
            SOURCES
                ipc.cpp
            ENVIRONMENT
                "UR_ADAPTERS_FORCE_LOAD=\"$<TARGET_FILE:ur_adapter_${adapter}>\""
        )

        target_link_libraries(test-adapter-${adapter}_ipc PRIVATE
            ur_umf
        )
    endif()

    add_adapter_test(${adapter}_mem_buffer_map
        FIXTURE DEVICES
        SOURCES
            urEnqueueMemBufferMapHostPtr.cpp
        ENVIRONMENT
            "UR_ADAPTERS_FORCE_LOAD=\"$<TARGET_FILE:ur_adapter_${adapter}>\""
        )

    add_adapter_test(${adapter}_mem_buffer_create_with_native_handle
        FIXTURE DEVICES
        SOURCES
            urMemBufferCreateWithNativeHandleShared.cpp
        ENVIRONMENT
            "UR_ADAPTERS_FORCE_LOAD=\"$<TARGET_FILE:ur_adapter_${adapter}>\""
        )
    target_link_libraries(test-adapter-${adapter}_mem_buffer_create_with_native_handle PRIVATE LevelZeroLoader LevelZeroLoader-Headers)
endfunction()

if(UR_BUILD_ADAPTER_L0)
    add_adapter_tests(level_zero)
endif()

if(UR_BUILD_ADAPTER_L0_V2)
    add_adapter_tests(level_zero_v2)
    add_subdirectory(v2)
endif()
