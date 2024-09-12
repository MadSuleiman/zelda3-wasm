cmake_minimum_required(VERSION 3.18 FATAL_ERROR)

project(zelda3
        VERSION 0.0.1
        DESCRIPTION "Zelda3 PC Port"
        HOMEPAGE_URL "https://github.com/snesrev/zelda3"
        LANGUAGES C)

if (CMAKE_BINARY_DIR EQUAL CMAKE_SOURCE_DIR)
    message(WARNING "You are compiling in the source tree, you probably don't want to do that. Use -B to set the build directory")
endif()

include(GNUInstallDirs)

# Compiler Warnings
list(APPEND c_warnings
            "-Wall"
            "-fno-strict-aliasing")

list(APPEND msvc_warnings
            "/W3"
            "/wd4996")
			
if( ${CMAKE_SYSTEM_NAME} MATCHES "Emscripten")
    set(USE_FLAGS "-fno-rtti -fno-exceptions -Wimplicit-function-declaration -O3 -sUSE_SDL=2")
	set(CMAKE_CXX_FLAGS_INIT "-sWASM=1 -sUSE_SDL=2")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${USE_FLAGS}")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${USE_FLAGS}")
	set(CMAKE_EXE_LINKER_FLAGS_INIT "-sWASM=1 -sUSE_SDL=2")
    set(CMAKE_EXE_LINKER "${CMAKE_EXE_LINKER_FLAGS_INIT} -s EXTRA_EXPORTED_RUNTIME_METHODS='[\"ccall\", \"cwrap\"]'")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${USE_FLAGS}")

	include_directories(
		${EMSCRIPTEN_ROOT_PATH}/system
		${CMAKE_SOURCE_DIR}/include
		${SDL2_INCLUDE_DIRS}
	)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --embed-file ${CMAKE_SOURCE_DIR}/zelda3_assets.dat@/ --embed-file ${CMAKE_SOURCE_DIR}/zelda3.ini@/")
	set_target_properties(${target_name} PROPERTIES LINK_DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/zelda3.sfc)
else()
	if (NINTENDO_SWITCH)
		# Force static on Switch, just because it's easier
		find_package(SDL2 REQUIRED COMPONENTS SDL2-static)
	else()
		find_package(SDL2 REQUIRED COMPONENTS SDL2)
	endif()
endif()

# For installation later
set(ini_name "${PROJECT_SOURCE_DIR}/zelda3.ini")

# Dependencies
# ------------
# SDL


# OpenGL
# TODO: This should likely be removed
add_library(gl STATIC "third_party/gl_core/gl_core_3_1.c")
add_library(opus STATIC "third_party/opus-1.3.1-stripped/opus_decoder_amalgam.c")
target_include_directories(gl PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}")

# Main Executable
# ---------------
add_executable(zelda3)
add_subdirectory(src)
add_subdirectory(snes)

# Math library
find_library(MATHLIB m)

# Target properties
target_link_libraries(zelda3 PRIVATE
                      $<$<BOOL:${MATHLIB}>:m>
                      SDL2 gl opus idbfs.js)

set_target_properties(zelda3 PROPERTIES C_STANDARD 11)

target_compile_options(zelda3 PRIVATE
                       $<IF:$<C_COMPILER_ID:MSVC>,${msvc_warnings},${c_warnings}>)

target_compile_definitions(zelda3 PRIVATE
                          $<$<NOT:$<C_COMPILER_ID:MSVC>>:SYSTEM_VOLUME_MIXER_AVAILABLE=0>
                          $<$<BOOL:${NINTENDO_SWITCH}>:__SWITCH__>)

# Nintendo Switch extra setup
if (NINTENDO_SWITCH)
    # needs to be linked with g++ for C++ stdlib
    enable_language(CXX)
    set_target_properties(zelda3 PROPERTIES
                          CXX_STANDARD 11
                          LINKER_LANGUAGE CXX)

    nx_generate_nacp(zelda3.nacp
                     NAME "Zelda 3"
                     AUTHOR "snesrev"
                     VERSION "${PROJECT_VERSION}")
    
    nx_create_nro(zelda3
                  NACP zelda3.nacp
                  ICON "${PROJECT_SOURCE_DIR}/src/platform/switch/icon.jpg")

    set(ini_name "${PROJECT_SOURCE_DIR}/src/platform/switch/zelda3.ini")
endif()

# Installation
# ------------
install(TARGETS zelda3 RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}")

# TODO: should be in some config dir
install(FILES "${ini_name}" DESTINATION "${CMAKE_INSTALL_BINDIR}")
