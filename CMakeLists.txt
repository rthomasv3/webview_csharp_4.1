cmake_minimum_required(VERSION 3.16)
project(webview VERSION 0.10.0)

set(CMAKE_CXX_STANDARD 17)

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
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} -framework WebKit")
endif()

include(FetchContent)
FetchContent_Declare(
    webviewNative
    GIT_REPOSITORY https://github.com/webview/webview
    GIT_TAG 0.12.0
)

# Set options for webview build
set(WEBVIEW_BUILD_SHARED_LIBRARY ON CACHE BOOL "" FORCE)
set(WEBVIEW_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(WEBVIEW_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(WEBVIEW_BUILD_DOCS OFF CACHE BOOL "" FORCE)

FetchContent_MakeAvailable(webviewNative)

# Create an alias target for the webview shared library
add_library(webview ALIAS webview::core_shared)

# For Windows, copy the WebView2Loader.dll if it exists
if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/libs/WebView2Loader.dll)
        configure_file(${CMAKE_CURRENT_SOURCE_DIR}/libs/WebView2Loader.dll WebView2Loader.dll COPYONLY)
    endif()
endif()