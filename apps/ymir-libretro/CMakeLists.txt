if (LIBRETRO_STATIC)
    add_library(ymir_libretro STATIC
        src/libretro.cpp
    )
else ()
    add_library(ymir_libretro SHARED
        src/libretro.cpp
    )
endif ()

set_target_properties(ymir_libretro PROPERTIES
    OUTPUT_NAME "ymir_libretro${LIBRETRO_SUFFIX}"
    PREFIX ""
    POSITION_INDEPENDENT_CODE ON
)

target_include_directories(ymir_libretro PRIVATE
    include
)

target_link_libraries(ymir_libretro PRIVATE
    ymir::ymir-core
)

target_compile_features(ymir_libretro PUBLIC cxx_std_20)

if (NOT MSVC)
    target_compile_options(ymir_libretro PRIVATE -fvisibility=hidden)
endif ()

if (IPO_SUPPORTED AND Ymir_ENABLE_IPO)
    set_property(TARGET ymir_libretro PROPERTY INTERPROCEDURAL_OPTIMIZATION TRUE)
endif ()

# For static libretro builds (e.g. Switch), bundle all dependency archives into
# one fat .a so RetroArch can link against a single file.
if (LIBRETRO_STATIC)
    set(_bundled_output "$<TARGET_FILE_DIR:ymir_libretro>/$<TARGET_FILE_NAME:ymir_libretro>")
    set(_merge_script "${CMAKE_CURRENT_BINARY_DIR}/merge_static_libs.sh")
    file(GENERATE OUTPUT "${_merge_script}" CONTENT
"#!/bin/bash
set -e
${CMAKE_AR} -M <<'MRI'
CREATE ${_bundled_output}
ADDLIB $<TARGET_FILE:ymir_libretro>
ADDLIB $<TARGET_FILE:ymir-core>
ADDLIB $<TARGET_FILE:fmt>
ADDLIB $<TARGET_FILE:libzstd_static>
ADDLIB $<TARGET_FILE:chdr-static>
ADDLIB $<TARGET_FILE:lzma>
ADDLIB $<TARGET_FILE:xxHash>
SAVE
END
MRI
${CMAKE_RANLIB} ${_bundled_output}
")
    add_custom_command(TARGET ymir_libretro POST_BUILD
        COMMAND bash "${_merge_script}"
        COMMENT "Bundling static libraries into ${_bundled_output}"
    )
endif ()
