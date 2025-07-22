if(CONFIG_APPTRACE_GCOV_ENABLE)
    if(CMAKE_C_COMPILER_ID MATCHES "Clang")
        # TODO: LLVM-214
        message(FATAL_ERROR "Coverage info is not supported when building with Clang!")
    endif()

    # Set a name for Gcov library
    set(GCOV_LIB libgcov_rtio)

    # Set include directory of Gcov internal headers
    execute_process(COMMAND ${CMAKE_C_COMPILER} -print-file-name=plugin
                    OUTPUT_VARIABLE gcc_plugin_dir
                    OUTPUT_STRIP_TRAILING_WHITESPACE
                    ERROR_QUIET
                    RESULT_VARIABLE gcc_plugin_result)
    
    set_source_files_properties(gcov_rtio.c
                                PROPERTIES COMPILE_FLAGS "-I${gcc_plugin_dir}/include")

    # Copy libgcov.a with symbols redefinition
    find_library(GCOV_LIBRARY_PATH gcov ${CMAKE_C_IMPLICIT_LINK_DIRECTORIES})
    
    # Check if libgcov.a was found
    if(NOT GCOV_LIBRARY_PATH)
        message(FATAL_ERROR "GCOV: libgcov.a not found in system directories: ${CMAKE_C_IMPLICIT_LINK_DIRECTORIES}")
    endif()
    
    # Verify libgcov.a file exists and is readable
    if(NOT EXISTS "${GCOV_LIBRARY_PATH}")
        message(FATAL_ERROR "GCOV: libgcov.a file does not exist: ${GCOV_LIBRARY_PATH}")
    endif()
    
    # Check if symbol map file exists
    set(SYMBOL_MAP_FILE "${CMAKE_CURRENT_LIST_DIR}/io_sym.map")
    if(NOT EXISTS "${SYMBOL_MAP_FILE}")
        message(FATAL_ERROR "GCOV: Symbol map file not found: ${SYMBOL_MAP_FILE}")
    endif()
    
    # Check if objcopy tool exists
    find_program(OBJCOPY_TOOL "${_CMAKE_TOOLCHAIN_PREFIX}objcopy")
    if(NOT OBJCOPY_TOOL)
        message(FATAL_ERROR "GCOV: objcopy tool not found: ${_CMAKE_TOOLCHAIN_PREFIX}objcopy. "
                           "Make sure your toolchain is properly installed.")
    endif()
    
    add_custom_command(OUTPUT ${GCOV_LIB}.a
                       COMMAND ${OBJCOPY_TOOL}
                               --redefine-syms ${SYMBOL_MAP_FILE}
                               ${GCOV_LIBRARY_PATH} ${GCOV_LIB}.a
                       COMMAND ${CMAKE_COMMAND} -E echo "GCOV: Successfully created modified library: ${GCOV_LIB}.a"
                       MAIN_DEPENDENCY ${GCOV_LIBRARY_PATH}
                       DEPENDS ${SYMBOL_MAP_FILE}
                       VERBATIM
                       COMMENT "Creating GCOV library with symbol redefinition")
    
    add_custom_target(${GCOV_LIB}_target DEPENDS ${GCOV_LIB}.a)
    add_library(${GCOV_LIB} STATIC IMPORTED)
    set_target_properties(${GCOV_LIB}
                          PROPERTIES
                          IMPORTED_LOCATION ${CMAKE_CURRENT_BINARY_DIR}/${GCOV_LIB}.a)
    add_dependencies(${GCOV_LIB} ${GCOV_LIB}_target)

    # Register component with app_trace dependency
    idf_component_register(SRCS "gcov_rtio.c"
                          REQUIRES "app_trace"
                          INCLUDE_DIRS ".")
    
    # Verify that gcov_rtio.c source file exists
    if(NOT EXISTS "${CMAKE_CURRENT_LIST_DIR}/gcov_rtio.c")
        message(FATAL_ERROR "GCOV: Source file not found: ${CMAKE_CURRENT_LIST_DIR}/gcov_rtio.c")
    endif()

    # Configure gcov-specific flags
    target_compile_options(${COMPONENT_LIB} PRIVATE "-fno-profile-arcs" "-fno-test-coverage")
    target_link_options(${COMPONENT_LIB} INTERFACE "-Wl,--wrap=__gcov_init")
    target_link_libraries(${COMPONENT_LIB} INTERFACE ${GCOV_LIB} c)
    
    message(STATUS "GCOV: Component configured successfully")
    message(STATUS "GCOV: GCC Plugin Dir: ${gcc_plugin_dir}")
    message(STATUS "GCOV: Original Library: ${GCOV_LIBRARY_PATH}")
    message(STATUS "GCOV: Modified Library: ${CMAKE_CURRENT_BINARY_DIR}/${GCOV_LIB}.a")

else()
    # Register empty component when GCOV is disabled
    idf_component_register(REQUIRES "app_trace")
    message(STATUS "GCOV: Component registered but GCOV support is disabled")
endif()
