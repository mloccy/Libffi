cmake_minimum_required(VERSION 3.20)
project(ffi C)
enable_language(ASM_MASM)
set(VERSION 3.2.1)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(FFI_CLOSURES 1)
set(FFI_DEBUG 0)
set(FFI_NO_STRUCTS 0)
set(FFI_NO_RAW_API 0)
set(FFI_EXEC_TRAMPOLINE_TABLE 0)

set(TARGET X86)
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
set(bin "${CMAKE_CURRENT_SOURCE_DIR}/build")
add_library(${PROJECT_NAME} STATIC)

if (MSVC)
    set(SYMBOL_UNDERSCORE 1)
    set(HAVE_HIDDEN_VISIBILITY_ATTRIBUTE 0)
    set(sources
        "${src}/closures.c"
        "${src}/prep_cif.c"
        "${src}/raw_api.c"
        "${src}/types.c"
    )

    if(HAVE_64BIT)
        set(TARGET X86_WIN64)
        set(HAVE_AS_X86_64_UNWIND_SECTION_TYPE 1)
        list(APPEND sources "${src}/ffiw64.c")
        configure_file("${src}/asm/win64.S" "${bin}/source/win64.S.i" COPYONLY)
        set(pre "${bin}/source/win64.S.i")
        set(asm "${bin}/source/win64.S")
        set(fmasm "/c /Cx /Fo")
    else()
        set(TARGET X86_WIN32)
        set(HAVE_AS_X86_64_UNWIND_SECTION_TYPE 0)
        list(APPEND sources "${src}/ffi.c")

        configure_file("${src}/asm/win32.S" "${bin}/source/win32.S.i" COPYONLY)
        set(pre "${bin}/source/win32.S.i")
        set(asm "${bin}/source/win32.S")
        set(fmasm "/c /Cx /coff /safeseh /Fo")
    endif()

    target_sources(${PROJECT_NAME} PRIVATE ${sources})

    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /Oy- /Oi /Gy /Zi /Gd")
    set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} /Oy-")

    add_custom_command(OUTPUT ${asm}
                       COMMAND ${CMAKE_C_COMPILER} ${pre} /I ${bin}/include /EP > ${asm}
                       DEPENDS ${pre})
    set_property(SOURCE ${asm} APPEND PROPERTY OBJECT_DEPENDS ${asm})
    add_custom_target(preprocess DEPENDS ${asm})
    target_sources(preprocess PRIVATE ${pre})
endif()

target_include_directories(${PROJECT_NAME} PRIVATE "${bin}/include")
configure_file("${inc}/ffi.h.in" "${bin}/include/ffi.h")
configure_file("${inc}/fficonfig.h.in" "${bin}/include/fficonfig.h")
configure_file("${inc}/ffitarget.h" "${bin}/include/ffitarget.h" COPYONLY)
configure_file("${inc}/ffi_common.h" "${bin}/include/ffi_common.h" COPYONLY)
configure_file("${inc}/tramp.h" "${bin}/include/tramp.h" COPYONLY)

set(PROCESSED_HEADERS "${bin}/include/ffi.h" "${bin}/include/fficonfig.h" "${bin}/include/ffitarget.h" "${bin}/include/ffi_common.h")
set_source_files_properties(${sources} PROPERTIES LANGUAGE C)
set_source_files_properties(${asm}
    PROPERTIES
        LANGUAGE ASM_MASM
        GENERATED TRUE
        COMPILE_FLAGS ${fmasm}
    )

# add_library(${PROJECT_NAME} ${sources} ${asm})
set(CMAKE_STATIC_LINKER_FLAGS "${CMAKE_STATIC_LINKER_FLAGS} /LTCG")
target_compile_definitions(${PROJECT_NAME} PUBLIC FFI_BUILDING FFI_STATIC_BUILD USE_DL_PREFIX)
set_target_properties(${PROJECT_NAME} PROPERTIES PREFIX "lib")

include(GNUInstallDirs)
install(TARGETS ${PROJECT_NAME} EXPORT ${PROJECT_NAME}Targets ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})
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

add_dependencies(${PROJECT_NAME} preprocess)
