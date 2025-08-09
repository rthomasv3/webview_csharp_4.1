cmake_minimum_required(VERSION 3.11.0)
project(webview VERSION 0.10.0)

set(CMAKE_CXX_STANDARD 17)

if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT -DWEBVIEW_STATIC")
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd -DWEBVIEW_STATIC")
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
    set(CMAKE_CXX_FLAGS "-DWEBVIEW_STATIC")
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    set(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} -framework WebKit -DWEBVIEW_STATIC")
endif()

# Download and extract the webview header directly
include(FetchContent)
FetchContent_Declare(
    webviewHeader
    URL https://github.com/webview/webview/archive/refs/tags/0.12.0.tar.gz
    URL_HASH SHA256=6b5b2c8e13287aa8e45afbe6c35ca8e37a3e1c5977e3c595070d7fda009c5bf1
)
FetchContent_MakeAvailable(webviewHeader)

# Create a simple wrapper source file that includes the amalgamated header
file(WRITE ${CMAKE_BINARY_DIR}/webview_wrapper.cc
"#define WEBVIEW_IMPLEMENTATION
#include \"webview.h\"
")

# Copy the header file to build directory
file(COPY ${webviewheader_SOURCE_DIR}/include/webview/webview.h 
     DESTINATION ${CMAKE_BINARY_DIR})

include_directories(
    "${CMAKE_BINARY_DIR}"
    "${CMAKE_CURRENT_SOURCE_DIR}/libs/include"
)
link_directories("${CMAKE_CURRENT_SOURCE_DIR}/libs")

add_library(webview SHARED ${CMAKE_BINARY_DIR}/webview_wrapper.cc)

if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    target_compile_definitions(webview PRIVATE "WEBVIEW_API=__declspec(dllexport)")
    target_link_libraries(webview WebView2Loader.dll.lib)
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/libs/WebView2Loader.dll WebView2Loader.dll COPYONLY)
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    target_link_libraries(webview PkgConfig::gtk3 PkgConfig::webkit2)
endif()