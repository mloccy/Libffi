cmake_minimum_required(VERSION 3.20)
project(ffi C ASM_MASM)

set(VERSION 3.2.1)
set(FFI_CLOSURES 1)
set(FFI_DEBUG 0)
set(FFI_NO_STRUCTS 0)
set(FFI_NO_RAW_API 0)
set(FFI_EXEC_TRAMPOLINE_TABLE 0)
set(FFI_CPU_ARCH X86)
set(FFI_HAVE_AS_CFI_PSEUDO_OP 0)

set(HAVE_LONG_DOUBLE 0)
set(HAVE_LONG_DOUBLE_VARIANT 0)

message(STATUS "Setting build type to '${CMAKE_BUILD_TYPE}'")
if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(HAVE_64BIT 1)
    set(HAVE_AS_X86_64_UNWIND_SECTION_TYPE 1)
    message(STATUS "Build configured for x86-64 architecture")
else()
    set(HAVE_64BIT 0)
    message(STATUS "Build configured for x86 architecture")
endif()

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_RELEASE "${CMAKE_BINARY_DIR}/lib")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_DEBUG "${CMAKE_BINARY_DIR}/lib")
set(inc "${CMAKE_CURRENT_SOURCE_DIR}/include")
set(src "${CMAKE_CURRENT_SOURCE_DIR}/source")
set(bin "${CMAKE_CURRENT_BINARY_DIR}")


function (CompileASM)
    set(options "")
    set(oneValueArgs ASM_FILE_PATH ASSEMBLER_PATH COMPILER_PATH ASSEMBLER_INCLUDE_PATH OUTPUT_OBJ_PATH )
    set(multiValueArgs ASSEMBLER_OPTIONS)

    cmake_parse_arguments(args "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

    get_filename_component(ASM_FILENAME ${args_ASM_FILE_PATH} NAME_WE)
    set(POSTPROC "${ASM_FILENAME}.s")
    set(POSTPROC_DIR "${CMAKE_CURRENT_BINARY_DIR}/postproc")

    if (NOT EXISTS ${POSTPROC_DIR})
        file(MAKE_DIRECTORY ${POSTPROC_DIR})
    endif()
    set(POSTPROC_PATH "${POSTPROC_DIR}/${POSTPROC}")

    add_custom_command(OUTPUT ${POSTPROC_PATH}
                       COMMAND ${args_COMPILER_PATH} /nologo ${args_ASM_FILE_PATH} /I "${args_ASSEMBLER_INCLUDE_PATH}" /EP > ${POSTPROC_PATH}
                       DEPENDS ${args_ASM_FILE_PATH})
    set(OUTPUT_DIR "${CMAKE_CURRENT_BINARY_DIR}/asm_objs")

    if (NOT EXISTS ${OUTPUT_DIR})
        file(MAKE_DIRECTORY ${OUTPUT_DIR})
    endif()

    set(OUTPUT_PATH "${OUTPUT_DIR}/${ASM_FILENAME}.obj")

    add_custom_command(OUTPUT ${OUTPUT_PATH}
                       COMMAND ${args_ASSEMBLER_PATH} /nologo /Fo "${OUTPUT_PATH}" /c ${POSTPROC_PATH} ${args_ASSEMBLER_OPTIONS}
                       DEPENDS ${POSTPROC_PATH})
    set_source_files_properties(${OUTPUT_PATH} PROPERTIES EXTERNAL_OBJECT TRUE)

    set(${args_OUTPUT_OBJ_PATH} ${OUTPUT_PATH} PARENT_SCOPE)
endfunction()

if (MSVC)
    set(SYMBOL_UNDERSCORE 1)
    set(HAVE_HIDDEN_VISIBILITY_ATTRIBUTE 0)
    set(sources
        "${src}/closures.c"
        "${src}/prep_cif.c"
        "${src}/raw_api.c"
        "${src}/tramp.c"
        "${src}/types.c"
    )

    if(HAVE_64BIT)
        set(FFI_CPU_ARCH X86_WIN64)
        set(HAVE_AS_X86_64_UNWIND_SECTION_TYPE 1)
        list(APPEND sources "${src}/ffiw64.c")
        configure_file("${src}/asm/win64_intel.S" "${bin}/source/win64_intel.S.i" COPYONLY)
        set(pre "${bin}/source/win64_intel.S.i")
        set(fmasm "/Cx")
    else()
        set(FFI_CPU_ARCH X86_WIN32)
        set(HAVE_AS_X86_64_UNWIND_SECTION_TYPE 0)
        list(APPEND sources "${src}/ffi.c")
        list(APPEND sources "${bin}/source/win32.S")

        configure_file("${src}/asm/win32.S" "${bin}/source/win32.S.i" COPYONLY)
        set(pre "${bin}/source/win32.S.i")
        set(fmasm "/Cx /coff /safeseh")
    endif()

    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /Oy- /Oi /Gy /Zi /Gd")
    set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} /Oy-")

    CompileASM(
        ASM_FILE_PATH ${pre}
        ASSEMBLER_PATH ${CMAKE_ASM_MASM_COMPILER}
        COMPILER_PATH ${CMAKE_C_COMPILER}
        ASSEMBLER_INCLUDE_PATH "${bin}/include"
        OUTPUT_OBJ_PATH ASM_OBJ
        ASSEMBLER_OPTIONS ${fmasm})
endif()

configure_file("${inc}/ffi.h.in" "${bin}/include/ffi.h")
configure_file("${inc}/fficonfig.h.in" "${bin}/include/fficonfig.h")
configure_file("${inc}/ffitarget.h" "${bin}/include/ffitarget.h" COPYONLY)
configure_file("${inc}/ffi_common.h" "${bin}/include/ffi_common.h" COPYONLY)
configure_file("${inc}/tramp.h" "${bin}/include/tramp.h" COPYONLY)
configure_file("${inc}/ffi_cfi.h" "${bin}/include/ffi_cfi.h" COPYONLY)
configure_file("${inc}/asmnames.h" "${bin}/include/asmnames.h" COPYONLY)

set(PROCESSED_HEADERS "${bin}/include/ffi.h" "${bin}/include/fficonfig.h" "${bin}/include/ffitarget.h" "${bin}/include/ffi_common.h")

list(APPEND sources ${ASM_OBJ})


add_library(${PROJECT_NAME} STATIC ${sources})
target_include_directories(${PROJECT_NAME} PRIVATE "${bin}/include")

set(CMAKE_STATIC_LINKER_FLAGS "${CMAKE_STATIC_LINKER_FLAGS} /LTCG")
target_compile_definitions(${PROJECT_NAME} PUBLIC FFI_BUILDING FFI_STATIC_BUILD USE_DL_PREFIX)
set_target_properties(${PROJECT_NAME} PROPERTIES PREFIX "lib")

include(GNUInstallDirs)
install(TARGETS ${PROJECT_NAME}  EXPORT ${PROJECT_NAME}Targets ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(FILES ${PROCESSED_HEADERS} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

install(
  EXPORT ${PROJECT_NAME}Targets
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
  NAMESPACE ${PROJECT_NAME}::
  FILE ${PROJECT_NAME}Targets.cmake
  )

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Version.cmake"
  VERSION ${VERSION}
  COMPATIBILITY AnyNewerVersion)

configure_package_config_file(
    "cmake/Config.cmake.in"
    "${PROJECT_NAME}Config.cmake"
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
    PATH_VARS CMAKE_INSTALL_LIBDIR)
install(
  FILES "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
        "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Version.cmake"
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
  )

