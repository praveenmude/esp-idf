if(BOOTLOADER_BUILD)
    # bootloader only needs FreeRTOS for config, not for anything else
    idf_component_register()
    return()
endif()

set(srcs
    "xtensa/port.c"
    "xtensa/portasm.S"
    "xtensa/xtensa_context.S"
    "xtensa/xtensa_init.c"
    "xtensa/xtensa_overlay_os_hook.c"
    "xtensa/xtensa_vector_defaults.S"
    "xtensa/xtensa_vectors.S")

list(APPEND srcs 
    "croutine.c"
    "event_groups.c"
    "list.c"
    "queue.c"
    "tasks.c"
    "timers.c"
    "stream_buffer.c"
    "FreeRTOS-openocd.c")

set(include_dirs 
    include
    xtensa/include)

set(private_include_dirs
    include/freertos
    xtensa/include/freertos
    xtensa
    .)

if(CONFIG_ESP32_IRAM_AS_8BIT_ACCESSIBLE_MEMORY)
     list(APPEND srcs "xtensa/xtensa_loadstore_handler.S")
 endif()

# app_trace is required by FreeRTOS headers only when CONFIG_SYSVIEW_ENABLE=y,
# but requirements can't depend on config options, so always require it.
idf_component_register(SRCS "${srcs}"
                    INCLUDE_DIRS ${include_dirs}
                    PRIV_INCLUDE_DIRS  ${private_include_dirs} 
                    LDFRAGMENTS linker.lf
                    REQUIRES app_trace esp_timer
                    PRIV_REQUIRES soc)
    
idf_component_get_property(COMPONENT_DIR freertos COMPONENT_DIR)
idf_component_set_property(freertos ORIG_INCLUDE_PATH "${COMPONENT_DIR}/include/freertos/")

if(CONFIG_FREERTOS_DEBUG_OCDAWARE)
    target_link_libraries(${COMPONENT_LIB} INTERFACE "-Wl,--undefined=uxTopUsedPriority")
endif()

set_source_files_properties(
    tasks.c
    event_groups.c
    timers.c
    queue.c
    stream_buffer.c
    PROPERTIES COMPILE_DEFINITIONS
    _ESP_FREERTOS_INTERNAL
    )

# The freertos component provides the `start_app` and `start_app_other_cores` 
# if it is included in the build. It then calls `app_main` 
# from the main task created, which must be provided by the user. 
# Like for `start_app` and `start_app_other_cores`,
# we can't establish dependency on what we don't yet know, so we force the
# linker to not drop this symbol.
target_link_libraries(${COMPONENT_LIB} INTERFACE "-u app_main")