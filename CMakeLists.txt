cmake_minimum_required(VERSION 3.10)

project(ft2-clone VERSION 1.95.0.0 LANGUAGES C CXX DESCRIPTION "Fasttracker II clone")

if(WIN32)
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${ft2-clone_SOURCE_DIR}/release/win")
elseif(UNIX)
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${ft2-clone_SOURCE_DIR}/release/linux")
elseif(APPLE)
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${ft2-clone_SOURCE_DIR}/release/macos")
else()
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${ft2-clone_SOURCE_DIR}/release/other")
endif()

option(EXTERNAL_LIBFLAC "Use external FLAC library" OFF)
option(SDL2_STATIC "Use static SDL2 library" ON)

file(GLOB ft2-clone_SRC
    "${ft2-clone_SOURCE_DIR}/src/rtmidi/*.cpp"
    "${ft2-clone_SOURCE_DIR}/src/*.c"
    "${ft2-clone_SOURCE_DIR}/src/gfxdata/*.c"
    "${ft2-clone_SOURCE_DIR}/src/mixer/*.c"
    "${ft2-clone_SOURCE_DIR}/src/scopes/*.c"
    "${ft2-clone_SOURCE_DIR}/src/modloaders/*.c"
    "${ft2-clone_SOURCE_DIR}/src/smploaders/*.c"
)

if(WIN32)
    configure_file("${ft2-clone_SOURCE_DIR}/src/ft2-clone.rc.in" "${ft2-clone_SOURCE_DIR}/src/ft2-clone.rc")
    list(APPEND ft2-clone_SRC "${ft2-clone_SOURCE_DIR}/src/ft2-clone.rc")
endif()

add_executable(ft2-clone ${ft2-clone_SRC})

find_package(SDL2 REQUIRED)

target_include_directories(ft2-clone SYSTEM
    PRIVATE ${SDL2_INCLUDE_DIRS}
)

target_link_libraries(ft2-clone
        PRIVATE ${SDL_LIBRARIES}
)

if (SDL2_STATIC)
    message(STATUS "Using SDL2 static library")
    target_link_libraries(ft2-clone
        PRIVATE SDL2::SDL2-static
    )
else()
    target_link_libraries(ft2-clone
        PRIVATE SDL2::SDL2
    )
endif()

# Windows needs the SDL2main linked as well
if(WIN32)
    target_link_libraries(ft2-clone
    PRIVATE SDL2::SDL2main
    )
endif()

find_package(Threads REQUIRED)

target_link_libraries(ft2-clone
    PRIVATE Threads::Threads
)

# Set some MSVC specific flags for the build
if(WIN32)
    if(MSVC)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /W4 /MP /Oi /Ot /Oy /GF /Gm- /MT /GS- /Gy /fp:fast /Zc:wchar_t- /GR-")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4 /MP /Oi /Ot /Oy /GF /Gm- /MT /GS- /Gy /fp:fast /Zc:wchar_t- /GR-")
        target_link_options(ft2-clone
        PRIVATE /OPT:ICF /INCREMENTAL:NO /LARGEADDRESSAWARE /FIXED:NO /OPT:REF /SUBSYSTEM:WINDOWS
        )
        target_compile_definitions(ft2-clone
        PRIVATE _CRT_SECURE_NO_WARNINGS UNICODE _UNICODE
        )
    endif()
endif()

target_compile_definitions(ft2-clone
    PRIVATE HAS_MIDI
    PRIVATE HAS_LIBFLAC
)

# Add some specific MSVC links for Windows
if(MSVC)
    target_link_options(ft2-clone
        PRIVATE /NODEFAULTLIB:LIBCMT
    )
    target_link_libraries(ft2-clone
    PRIVATE odbc32.lib odbccp32.lib winmm.lib shlwapi.lib
    )
    target_compile_definitions(ft2-clone
    PRIVATE __WINDOWS_MM__
    )
endif()

if(UNIX)
    if(APPLE)
        find_library(COREAUDIO CoreAudio REQUIRED)
        find_library(COREFOUNDATION CoreFoundation REQUIRED)
        find_library(COREMIDI CoreMidi REQUIRED)
        find_library(ICONV iconv REQUIRED)
        target_link_libraries(ft2-clone
            PRIVATE ${COREAUDIO} ${COREMIDI} ${COREFOUNDATION} ${ICONV}
        )
        target_compile_definitions(ft2-clone
            PRIVATE __MACOSX_CORE__
        )
    else()
        target_link_libraries(ft2-clone
        PRIVATE asound
        )
        target_compile_definitions(ft2-clone
            PRIVATE __LINUX_ALSA__
        )
    endif()
endif()

if(EXTERNAL_LIBFLAC)
    find_package(Flac REQUIRED)
    target_compile_definitions(ft2-clone
        PRIVATE EXTERNAL_LIBFLAC
    )
    target_link_libraries(ft2-clone
    PRIVATE FLAC::FLAC
    )
else()
    file(GLOB flac_SRC
        "${ft2-clone_SOURCE_DIR}/src/libflac/*.c"
    )
    target_sources(ft2-clone
        PRIVATE ${flac_SRC}
    )
endif()

install(TARGETS ft2-clone
    RUNTIME DESTINATION bin
)