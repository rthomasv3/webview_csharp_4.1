cmake_minimum_required(VERSION 3.16)
project(webview VERSION 0.10.0)

set(CMAKE_CXX_STANDARD 17)

# Platform-specific flags
if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT")
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd")
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(
        gtk3 REQUIRED gtk+-3.0
        IMPORTED_TARGET
      )
    pkg_check_modules(
        webkit2 REQUIRED webkit2gtk-4.1
        IMPORTED_TARGET
      )
endif()

include(FetchContent)
FetchContent_Declare(
    webviewNative
    GIT_REPOSITORY https://github.com/webview/webview
    GIT_TAG 0.12.0
)

# Configure webview build options
set(WEBVIEW_BUILD_SHARED_LIBRARY ON CACHE BOOL "" FORCE)
set(WEBVIEW_BUILD_STATIC_LIBRARY OFF CACHE BOOL "" FORCE)
set(WEBVIEW_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(WEBVIEW_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(WEBVIEW_BUILD_DOCS OFF CACHE BOOL "" FORCE)

FetchContent_MakeAvailable(webviewNative)

# The webview project creates webview_shared target
# We'll create an alias to maintain compatibility with existing C# bindings
if(TARGET webview_shared)
    add_library(webview ALIAS webview_shared)
else()
    message(FATAL_ERROR "webview_shared target not found after FetchContent_MakeAvailable")
endif()

# For Windows, copy the WebView2Loader.dll if it exists
if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/libs/WebView2Loader.dll)
        configure_file(${CMAKE_CURRENT_SOURCE_DIR}/libs/WebView2Loader.dll ${CMAKE_BINARY_DIR}/WebView2Loader.dll COPYONLY)
    endif()
endif()