cmake_minimum_required(VERSION 3.11.0)
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
FetchContent_Populate(webviewNative)

# Include directories - only include what actually exists
include_directories(
    "${webviewnative_SOURCE_DIR}/core/include"
    "${webviewnative_SOURCE_DIR}/core/include/webview"
)

# Build the shared library from core/src/webview.cc
add_library(webview SHARED "${webviewnative_SOURCE_DIR}/core/src/webview.cc")

# Platform-specific definitions and linking
if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    target_compile_definitions(webview PRIVATE "WEBVIEW_API=__declspec(dllexport)")
    # Windows system libraries needed for WebView2
    target_link_libraries(webview advapi32 ole32 shell32 shlwapi user32 version)
    
    # Note: WebView2Loader.dll needs to be handled separately - either:
    # 1. Downloaded from NuGet during build
    # 2. Copied from the WebView2 SDK
    # 3. Use the built-in implementation (WEBVIEW_MSWEBVIEW2_BUILTIN_IMPL)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    target_link_libraries(webview PkgConfig::gtk3 PkgConfig::webkit2)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    target_link_libraries(webview "-framework WebKit")
endif()