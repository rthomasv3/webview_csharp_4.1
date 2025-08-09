cmake_minimum_required(VERSION 3.16)
project(webview VERSION 0.10.0)

set(CMAKE_CXX_STANDARD 17)

# Fetch webview
include(FetchContent)
FetchContent_Declare(
    webviewNative
    GIT_REPOSITORY https://github.com/webview/webview
    GIT_TAG 0.12.0
)

# Set webview build options before FetchContent_MakeAvailable
set(WEBVIEW_BUILD_SHARED_LIBRARY ON CACHE BOOL "" FORCE)
set(WEBVIEW_BUILD_STATIC_LIBRARY OFF CACHE BOOL "" FORCE)
set(WEBVIEW_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(WEBVIEW_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(WEBVIEW_BUILD_DOCS OFF CACHE BOOL "" FORCE)

# For Windows, download WebView2 SDK if needed
if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    # Download WebView2 SDK
    FetchContent_Declare(
        webview2
        URL https://www.nuget.org/api/v2/package/Microsoft.Web.WebView2/1.0.2210.55
        URL_HASH SHA256=5a7b87ba50fb59ac2db2c59b6fd3bc8971a2e98797fb946f98e1f8b3f6c0de7c
    )
    FetchContent_MakeAvailable(webview2)
    
    # Set include path for WebView2
    set(WEBVIEW_MSWEBVIEW2_INCLUDE_DIR "${webview2_SOURCE_DIR}/build/native/include" CACHE PATH "" FORCE)
    set(WEBVIEW_MSWEBVIEW2_VERSION "1.0.2210.55" CACHE STRING "" FORCE)
endif()

# Now make webview available - it will use the WebView2 paths we set
FetchContent_MakeAvailable(webviewNative)

# The webview project should have created webview_shared target
# Create an alias for compatibility with the C# bindings
if(TARGET webview_shared)
    # Create a custom target that ensures proper naming
    add_library(webview ALIAS webview_shared)
    
    # On Windows, ensure the output is named webview.dll (not libwebview.dll)
    if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
        set_target_properties(webview_shared PROPERTIES
            OUTPUT_NAME "webview"
            PREFIX ""
        )
        
        # Copy WebView2Loader.dll to output directory if it exists
        if(EXISTS "${webview2_SOURCE_DIR}/build/native/x64/WebView2Loader.dll")
            configure_file(
                "${webview2_SOURCE_DIR}/build/native/x64/WebView2Loader.dll"
                "${CMAKE_BINARY_DIR}/WebView2Loader.dll"
                COPYONLY
            )
        endif()
    endif()
else()
    message(FATAL_ERROR "webview_shared target was not created by the webview project")
endif()

# For Linux, ensure we're linking against webkit2gtk-4.1 (not 4.0)
if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    # The webview project should already handle this, but let's make sure
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(webkit2 REQUIRED webkit2gtk-4.1)
endif()