cmake_minimum_required (VERSION 2.8.11)

set(LAYER_JSON_FILES
    VkLayer_wrap_objects
    VkLayer_test
    )

set(VK_LAYER_RPATH /usr/lib/x86_64-linux-gnu/vulkan/layer:/usr/lib/i386-linux-gnu/vulkan/layer)
set(CMAKE_INSTALL_RPATH ${VK_LAYER_RPATH})

if (WIN32)
    if (NOT (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR))
        foreach (config_file ${LAYER_JSON_FILES})
            FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/windows/${config_file}.json src_json)
            if (CMAKE_GENERATOR MATCHES "^Visual Studio.*")
                FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_BINARY_DIR}/$<CONFIGURATION>/${config_file}.json dst_json)
            else()
                FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_BINARY_DIR}/${config_file}.json dst_json)
            endif()
            add_custom_target(${config_file}-json ALL
                COMMAND copy ${src_json} ${dst_json}
                VERBATIM
                )
                add_dependencies(${config_file}-json ${config_file})
        endforeach(config_file)
    endif()
else()
    # extra setup for out-of-tree builds
    if (NOT (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR))
        foreach (config_file ${LAYER_JSON_FILES})
            add_custom_target(${config_file}-json ALL
                COMMAND ln -sf ${CMAKE_CURRENT_SOURCE_DIR}/linux/${config_file}.json
                VERBATIM
                )
        endforeach(config_file)
    endif()
endif()

if (WIN32)
    macro(add_vk_layer target)
    add_custom_command(OUTPUT VkLayer_${target}.def
        COMMAND ${PYTHON_CMD} ${SCRIPTS_DIR}/vk-generate.py ${DisplayServer} win-def-file VkLayer_${target} layer > VkLayer_${target}.def
        DEPENDS ${SCRIPTS_DIR}/vk-generate.py ${SCRIPTS_DIR}/vulkan.py
    )
    add_library(VkLayer_${target} SHARED ${ARGN} VkLayer_${target}.def)
    add_dependencies(VkLayer_${target} generate_tests_dispatch_table_helper VkLayer_utils)
    endmacro()
else()
    macro(add_vk_layer target)
    add_library(VkLayer_${target} SHARED ${ARGN})
    add_dependencies(VkLayer_${target} generate_tests_dispatch_table_helper VkLayer_utils)
    set_target_properties(VkLayer_${target} PROPERTIES LINK_FLAGS "-Wl,-Bsymbolic")
    endmacro()
endif()

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/../../layers
    ${CMAKE_CURRENT_SOURCE_DIR}/../../loader
    ${CMAKE_CURRENT_SOURCE_DIR}/../../include/vulkan
    ${CMAKE_CURRENT_BINARY_DIR}
    ${CMAKE_CURRENT_BINARY_DIR}/../../layers
)

if (WIN32)
    set (CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -D_CRT_SECURE_NO_WARNINGS")
    set (CMAKE_C_FLAGS_RELEASE   "${CMAKE_C_FLAGS_RELEASE} -D_CRT_SECURE_NO_WARNINGS")
    set (CMAKE_CXX_FLAGS_DEBUG   "${CMAKE_CXX_FLAGS_DEBUG} -D_CRT_SECURE_NO_WARNINGS /bigobj")
    set (CMAKE_C_FLAGS_DEBUG     "${CMAKE_C_FLAGS_DEBUG} -D_CRT_SECURE_NO_WARNINGS /bigobj")
else()
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wpointer-arith -Wno-unused-function")
    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wpointer-arith -Wno-unused-function")
endif()

add_custom_command(OUTPUT vk_dispatch_table_helper.h
    COMMAND ${PYTHON_CMD} ${SCRIPTS_DIR}/vk-generate.py AllPlatforms dispatch-table-ops layer > vk_dispatch_table_helper.h
    DEPENDS ${SCRIPTS_DIR}/vk-generate.py ${SCRIPTS_DIR}/vulkan.py)

add_custom_target(generate_tests_dispatch_table_helper DEPENDS
    vk_dispatch_table_helper.h
)

set (WRAP_SRCS
       wrap_objects.cpp
	   ${CMAKE_CURRENT_SOURCE_DIR}/../../layers/vk_layer_table.cpp
	   ${CMAKE_CURRENT_SOURCE_DIR}/../../layers/vk_layer_extension_utils.cpp
	   )
add_vk_layer(wrap_objects ${WRAP_SRCS})

set (TEST_SRCS
       test.cpp
	   ${CMAKE_CURRENT_SOURCE_DIR}/../../layers/vk_layer_table.cpp
	   ${CMAKE_CURRENT_SOURCE_DIR}/../../layers/vk_layer_extension_utils.cpp
	   )
add_vk_layer(test ${TEST_SRCS})


