cmake_minimum_required(VERSION 3.16)
project(mapmap VERSION 0.6.3 LANGUAGES CXX)

# Modern C++ standard required for Qt5 and GStreamer bindings
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Automate Qt's meta-object, UI, and resource compilation
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTORCC ON)

# Locate Qt5 Modules (WebEngine explicitly excluded)
find_package(Qt5 COMPONENTS Core Gui Widgets Multimedia OpenGL Network Xml REQUIRED)
find_package(OpenGL REQUIRED)

# ---------------------------------------------------------
# Compile Translations (Must run BEFORE AutoRCC processes them)
# ---------------------------------------------------------
# Find the Qt lrelease tool (sometimes named lrelease-qt5 on Linux)
find_program(LRELEASE_EXECUTABLE NAMES lrelease-qt5 lrelease REQUIRED)

# Gather all .ts files
file(GLOB TS_FILES "translations/*.ts")

if(TS_FILES AND LRELEASE_EXECUTABLE)
    message(STATUS "Compiling Qt translations with ${LRELEASE_EXECUTABLE}...")
    execute_process(
        COMMAND ${LRELEASE_EXECUTABLE} ${TS_FILES}
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    )
endif()
# ---------------------------------------------------------

# Locate System Dependencies via pkg-config
find_package(PkgConfig REQUIRED)
pkg_check_modules(GST REQUIRED gstreamer-1.0 gstreamer-app-1.0 gstreamer-video-1.0 gstreamer-pbutils-1.0)
pkg_check_modules(LIBLO REQUIRED liblo)

# ---------------------------------------------------------
# Restore missing Qt Solutions assets (The Proper Fix)
# ---------------------------------------------------------
file(MAKE_DIRECTORY "${CMAKE_SOURCE_DIR}/src/gui/contrib/qtpropertybrowser/src/images")

# The official archived Qt Solutions upstream repository
set(QT_UPSTREAM_URL "https://raw.githubusercontent.com/qtproject/qt-solutions/master/qtpropertybrowser/src/images")

set(MISSING_CURSORS
    cursor-sizeb.png cursor-sizef.png cursor-sizeall.png
    cursor-sizeh.png cursor-sizev.png cursor-closedhand.png
    cursor-busy.png cursor-crosshair.png cursor-forbidden.png
    cursor-hand.png cursor-hsplit.png cursor-vsplit.png
    cursor-ibeam.png cursor-openhand.png cursor-up.png
    cursor-wait.png cursor-whatsthis.png cursor-arrow.png
    cursor-uparrow.png
)

foreach(CURSOR_FILE ${MISSING_CURSORS})
    set(LOCAL_FILE "${CMAKE_SOURCE_DIR}/src/gui/contrib/qtpropertybrowser/src/images/${CURSOR_FILE}")
    # Only download if we don't already have it
    if(NOT EXISTS "${LOCAL_FILE}")
        message(STATUS "Restoring authentic asset from Qt upstream: ${CURSOR_FILE}")
        file(DOWNLOAD "${QT_UPSTREAM_URL}/${CURSOR_FILE}" "${LOCAL_FILE}" STATUS DOWNLOAD_STATUS)
    endif()
endforeach()

# Gather source and resource files
file(GLOB_RECURSE SRC_FILES "src/*.cpp" "src/*.h")
list(FILTER SRC_FILES EXCLUDE REGEX "src/gui/contrib/qtpropertybrowser/examples/.*")
file(GLOB_RECURSE QRC_FILES "*.qrc")
list(FILTER QRC_FILES EXCLUDE REGEX "src/gui/contrib/qtpropertybrowser/examples/.*")

add_executable(${PROJECT_NAME} ${SRC_FILES} ${QRC_FILES})

# Get all subdirectories inside src/
file(GLOB_RECURSE SRC_DIRS LIST_DIRECTORIES true "src/*")
list(FILTER SRC_DIRS EXCLUDE REGEX "src/gui/contrib/.*")
list(FILTER SRC_DIRS INCLUDE REGEX ".*/$") # Only keep directories

target_include_directories(${PROJECT_NAME} PRIVATE
    ${GST_INCLUDE_DIRS}
    ${LIBLO_INCLUDE_DIRS}
    "src"
    "src/app"
    "src/control"
    "src/control/qosc" # Required for oscreceiver.h
    "src/core"
    "src/gui"
    "src/gui/contrib/qtpropertybrowser/src"
    "src/gui/contrib/qtpropertybrowser-extension"
    "src/media"
    "src/shape"
    "src/util"
)

target_link_libraries(${PROJECT_NAME} PRIVATE
    Qt5::Core 
    Qt5::Gui 
    Qt5::Widgets 
    Qt5::Multimedia 
    Qt5::OpenGL 
    Qt5::Network 
    Qt5::Xml
    OpenGL::GL
    ${GST_LIBRARIES}
    ${LIBLO_LIBRARIES}
)

# Standard Linux installation paths using GNUInstallDirs
include(GNUInstallDirs)
install(TARGETS ${PROJECT_NAME} DESTINATION ${CMAKE_INSTALL_BINDIR})
install(FILES resources/mapmap.desktop DESTINATION ${CMAKE_INSTALL_DATADIR}/applications)
install(FILES resources/images/mapmap.png DESTINATION ${CMAKE_INSTALL_DATADIR}/pixmaps)

