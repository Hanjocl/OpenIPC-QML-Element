cmake_minimum_required(VERSION 3.20)
project(OpenIPC_QML_Element)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)
set(CMAKE_INCLUDE_CURRENT_DIR ON)


# Custom importig things
if(CMAKE_BUILD_TYPE MATCHES "Debug")
    message("build debug")
    add_definitions(-DDEBUG_MODE)
endif()

file(GLOB_RECURSE SRC_LIST
        3rd/rtl8812au-monitor-pcap/*.c
        3rd/rtl8812au-monitor-pcap/*.cpp
        src/*.cpp
        src/*.c
        src/*.h
)
list(FILTER SRC_LIST EXCLUDE REGEX ".*/main\\.cpp$")
include_directories(src 3rd/rtl8812au-monitor-pcap/src 3rd/rtl8812au-monitor-pcap/hal)


# -----------------------------
# Library: CRSF_USB_Lib
# -----------------------------
set(LIB_NAME "OpenIPC_QML_Element_Lib")
add_library(${LIB_NAME} STATIC 
    ${SRC_LIST}
)
target_include_directories(${LIB_NAME} PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    ${CMAKE_CURRENT_SOURCE_DIR}/src/player
    ${CMAKE_CURRENT_SOURCE_DIR}/3rd/rtl8812au-monitor-pcap/src
    ${CMAKE_CURRENT_SOURCE_DIR}/3rd/rtl8812au-monitor-pcap/hal
)

### PACKAGES ###
# Package Config
find_package(PkgConfig REQUIRED)
# SDL2
find_package(SDL2 REQUIRED)

# FFmpeg
if(WIN32)
    find_package(FFmpeg REQUIRED COMPONENTS avcodec avformat avutil swscale)
    # include_directories(${FFMPEG_INCLUDE_DIRS})
elseif(UNIX)
    # First try find_package
    find_package(FFmpeg QUIET COMPONENTS avcodec avformat avutil swscale avfilter avdevice)

    if(FFmpeg_FOUND)
        message(STATUS "FFmpeg found via find_package")
        include_directories(${FFMPEG_INCLUDE_DIRS})
        target_link_libraries(${LIB_NAME} ${FFMPEG_LIBRARIES})
    else()
        # Fallback to pkg-config
        find_package(PkgConfig REQUIRED)
        pkg_check_modules(FFMPEG REQUIRED
            libavcodec
            libavformat
            libavutil
            libswscale
            libavfilter
            libavdevice
        )

        include_directories(${FFMPEG_INCLUDE_DIRS})
        link_directories(${FFMPEG_LIBRARY_DIRS})
        target_link_libraries(${LIB_NAME} ${FFMPEG_LIBRARIES})
    endif()
endif()

# libsodium
if(WIN32)
    find_package(unofficial-sodium CONFIG REQUIRED)
elseif(UNIX)
    pkg_check_modules(SODIUM REQUIRED libsodium)

    include_directories(${SODIUM_INCLUDE_DIRS})
    link_directories(${SODIUM_LIBRARY_DIRS})
endif()

# libusb
pkg_check_modules(LIBUSB REQUIRED IMPORTED_TARGET libusb-1.0)

# QT6
find_package(Qt6 REQUIRED COMPONENTS Quick Multimedia QuickControls2 Widgets)

add_executable(${PROJECT_NAME} 
    ${SRC_LIST}
    qml.qrc
    src/main.cpp
)

# link libraries
if(WIN32)
    target_link_libraries(${LIB_NAME}
        Qt6::QuickControls2
        Qt6::Quick
        Qt6::Widgets
        ${FFMPEG_LIBRARIES}
        ${SDL2_LIBRARIES}   
        PkgConfig::LIBUSB
        unofficial-sodium::sodium
    )
elseif(UNIX)    
    target_link_libraries(${LIB_NAME}
        Qt6::QuickControls2
        Qt6::Quick
        Qt6::Widgets
        ${FFMPEG_LIBRARIES}
        ${SDL2_LIBRARIES}
        PkgConfig::LIBUSB
        ${SODIUM_LIBRARIES}
        swresample
    )
endif()
target_link_libraries(${PROJECT_NAME} PUBLIC ${LIB_NAME})


# -----------------------------
# Windows deployment
# -----------------------------
if(WIN32)
    find_program(QT_WINDEPLOYQT_EXECUTABLE windeployqt HINTS ${Qt6_DIR}/../../../bin)
    
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${QT_WINDEPLOYQT_EXECUTABLE} 
            --qmldir ${CMAKE_CURRENT_SOURCE_DIR}/qml
            $<TARGET_FILE:${PROJECT_NAME}>
        COMMENT "Running windeployqt with custom options"
    )
endif()
