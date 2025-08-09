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
set(WEBVIEW_BUILD OFF CACHE BOOL "" FORCE)  # Don't build their targets
set(WEBVIEW_BUILD_AMALGAMATION OFF CACHE BOOL "" FORCE)
set(WEBVIEW_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(WEBVIEW_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(WEBVIEW_BUILD_DOCS OFF CACHE BOOL "" FORCE)

FetchContent_MakeAvailable(webviewNative)

# Build our own shared library from the webview sources
file(GLOB WEBVIEW_SOURCES 
    "${webviewnative_SOURCE_DIR}/src/*.cc"
    "${webviewnative_SOURCE_DIR}/src/*.c"
)

add_library(webview SHARED ${WEBVIEW_SOURCES})

# Include the webview headers
target_include_directories(webview PUBLIC 
    "${webviewnative_SOURCE_DIR}/include"
)

# Define WEBVIEW_BUILD_SHARED for building a shared library
target_compile_definitions(webview PRIVATE WEBVIEW_BUILD_SHARED)

if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    target_compile_definitions(webview PRIVATE "WEBVIEW_API=__declspec(dllexport)")
    # Find and link WebView2
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/libs/WebView2Loader.dll.lib)
        target_link_libraries(webview PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/libs/WebView2Loader.dll.lib)
    endif()
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/libs/WebView2Loader.dll)
        configure_file(${CMAKE_CURRENT_SOURCE_DIR}/libs/WebView2Loader.dll WebView2Loader.dll COPYONLY)
    endif()
    # Windows system libraries
    target_link_libraries(webview PUBLIC advapi32 ole32 shell32 shlwapi user32 version)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    target_compile_definitions(webview PRIVATE WEBVIEW_GTK)
    target_link_libraries(webview PUBLIC PkgConfig::gtk3 PkgConfig::webkit2 dl)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    target_compile_definitions(webview PRIVATE WEBVIEW_COCOA)
    target_link_libraries(webview PUBLIC "-framework WebKit" dl)
endif()