cmake_minimum_required(VERSION 3.16)
project(webview VERSION 0.10.0)

set(CMAKE_CXX_STANDARD 17)

# Platform-specific flags
if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT")
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd")
endif()

# Fetch webview using their documented approach
include(FetchContent)
FetchContent_Declare(
    webview
    GIT_REPOSITORY https://github.com/webview/webview
    GIT_TAG 0.12.0
)
FetchContent_MakeAvailable(webview)

# Create a minimal wrapper source file that uses the C API
file(WRITE ${CMAKE_BINARY_DIR}/webview_shim.c
"#include \"webview/webview.h\"

// This file exists solely to create a shared library from the webview static library
// The webview static library will be linked in
")

# Build our shared library that wraps webview::core
add_library(webview SHARED ${CMAKE_BINARY_DIR}/webview_shim.c)

# Link against webview::core (the static library)
target_link_libraries(webview PRIVATE webview::core)

# Export all symbols on Windows
if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    target_compile_definitions(webview PRIVATE "WEBVIEW_API=__declspec(dllexport)")
    # Windows system libraries if needed
    target_link_libraries(webview PRIVATE advapi32 ole32 shell32 shlwapi user32 version)
    
    # Copy WebView2Loader.dll if it exists
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/libs/WebView2Loader.dll)
        configure_file(${CMAKE_CURRENT_SOURCE_DIR}/libs/WebView2Loader.dll ${CMAKE_BINARY_DIR}/WebView2Loader.dll COPYONLY)
    endif()
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    # On Linux, we need to ensure we're linked against GTK and WebKit
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(gtk3 REQUIRED gtk+-3.0)
    pkg_check_modules(webkit2 REQUIRED webkit2gtk-4.1)
    
    target_link_libraries(webview PRIVATE ${gtk3_LIBRARIES} ${webkit2_LIBRARIES})
    target_include_directories(webview PRIVATE ${gtk3_INCLUDE_DIRS} ${webkit2_INCLUDE_DIRS})
    target_compile_options(webview PRIVATE ${gtk3_CFLAGS_OTHER} ${webkit2_CFLAGS_OTHER})
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    target_link_libraries(webview PRIVATE "-framework WebKit")
endif()