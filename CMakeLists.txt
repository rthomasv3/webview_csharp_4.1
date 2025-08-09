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

# Create a wrapper source file that references all the C API functions
# This ensures they're all included in the shared library
file(WRITE ${CMAKE_BINARY_DIR}/webview_exports.c
"#include \"webview/webview.h\"

// Reference all C API functions to ensure they're exported
void* webview_force_exports() {
    void* funcs[] = {
        (void*)webview_create,
        (void*)webview_destroy,
        (void*)webview_run,
        (void*)webview_terminate,
        (void*)webview_dispatch,
        (void*)webview_get_window,
        (void*)webview_set_title,
        (void*)webview_set_size,
        (void*)webview_navigate,
        (void*)webview_set_html,
        (void*)webview_init,
        (void*)webview_eval,
        (void*)webview_bind,
        (void*)webview_unbind,
        (void*)webview_return,
        (void*)webview_version,
        (void*)webview_get_native_handle
    };
    return funcs[0];
}
")

# Build our shared library
add_library(webview SHARED ${CMAKE_BINARY_DIR}/webview_exports.c)

# Link against webview::core (the static library)
target_link_libraries(webview PUBLIC webview::core)

# Platform-specific settings
if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    # Force export all symbols
    set_target_properties(webview PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
    target_compile_definitions(webview PRIVATE "WEBVIEW_API=__declspec(dllexport)")
    
    # Set output name to webview.dll (not libwebview.dll)
    set_target_properties(webview PROPERTIES PREFIX "")
    
    # Windows system libraries
    target_link_libraries(webview PUBLIC advapi32 ole32 shell32 shlwapi user32 version)
    
    # Copy WebView2Loader.dll if it exists
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/libs/WebView2Loader.dll)
        configure_file(${CMAKE_CURRENT_SOURCE_DIR}/libs/WebView2Loader.dll ${CMAKE_BINARY_DIR}/WebView2Loader.dll COPYONLY)
    endif()
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    # Force include all symbols from static library
    set_target_properties(webview PROPERTIES LINK_FLAGS "-Wl,--whole-archive")
    
    # Linux needs GTK and WebKit
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(gtk3 REQUIRED gtk+-3.0)
    pkg_check_modules(webkit2 REQUIRED webkit2gtk-4.1)
    
    target_link_libraries(webview PUBLIC ${gtk3_LIBRARIES} ${webkit2_LIBRARIES} dl pthread)
    target_include_directories(webview PUBLIC ${gtk3_INCLUDE_DIRS} ${webkit2_INCLUDE_DIRS})
    target_compile_options(webview PUBLIC ${gtk3_CFLAGS_OTHER} ${webkit2_CFLAGS_OTHER})
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    # Force include all symbols from static library
    set_target_properties(webview PROPERTIES LINK_FLAGS "-Wl,-force_load,$<TARGET_FILE:webview::core>")
    target_link_libraries(webview PUBLIC "-framework WebKit" dl pthread)
endif()

# Set shared library version
set_target_properties(webview PROPERTIES VERSION ${PROJECT_VERSION} SOVERSION 0)