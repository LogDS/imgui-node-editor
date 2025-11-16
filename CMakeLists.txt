cmake_minimum_required(VERSION 4.0)
project(imgui-node-editor)

# Enable solution folders in Visual Studio and Folders in Xcode
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set(IMGUI_NODE_EDITOR_ROOT_DIR ${CMAKE_CURRENT_LIST_DIR})
# Point CMake where to look for module files.
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_LIST_DIR}/misc/cmake-modules)

include_directories(include)

set(_Application_Sources
        include/imgui-node-editor/application.h
    src/application.cpp
    src/entry_point.cpp
        include/imgui-node-editor/imgui_extra_keys.h
        include/imgui-node-editor/config.h.in
        include/imgui-node-editor/setup.h
        include/imgui-node-editor/platform.h
    src/platform_win32.cpp
    src/platform_glfw.cpp
        include/imgui-node-editor/renderer.h
    src/renderer_dx11.cpp
    src/renderer_ogl3.cpp
        include/imgui-node-editor/builders.h
        include/imgui-node-editor/drawing.h
        include/imgui-node-editor/widgets.h
    src/builders.cpp
    src/drawing.cpp
    src/widgets.cpp
)

add_library(imgui-node-editor STATIC)

target_include_directories(imgui-node-editor PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include)

find_package(imgui REQUIRED)
find_package(stb_image REQUIRED)
find_package(ScopeGuard REQUIRED)
target_link_libraries(imgui-node-editor PUBLIC imgui)
target_link_libraries(imgui-node-editor PRIVATE stb_image ScopeGuard)

if (WIN32)
    list(APPEND _Application_Sources
        src/imgui_impl_dx11.cpp
            include/imgui-node-editor/imgui_impl_dx11.h
        src/imgui_impl_win32.cpp
            include/imgui-node-editor/imgui_impl_win32.h
    )

    set(_DXSDK_Dir  ${IMGUI_NODE_EDITOR_ROOT_DIR}/external/DXSDK)
    set(_DXSDK_Arch x86)
    if (${CMAKE_SIZEOF_VOID_P} EQUAL 8)
        set(_DXSDK_Arch x64)
    endif()

    add_library(dxerr STATIC ${_DXSDK_Dir}/src/dxerr.cpp)
    target_include_directories(dxerr PUBLIC "${_DXSDK_Dir}/include")
    set_property(TARGET dxerr PROPERTY FOLDER "external")

    add_library(d3dx11 UNKNOWN IMPORTED)
    set_target_properties(d3dx11 PROPERTIES
        IMPORTED_LOCATION               "${_DXSDK_Dir}/lib/${_DXSDK_Arch}/d3dx11.lib"
        IMPORTED_LOCATION_DEBUG         "${_DXSDK_Dir}/lib/${_DXSDK_Arch}/d3dx11d.lib"
        INTERFACE_INCLUDE_DIRECTORIES   "${_DXSDK_Dir}/include"
        INTERFACE_LINK_LIBRARIES        "$<$<CONFIG:Debug>:dxerr>"
    )

    target_link_libraries(imgui-node-editor PRIVATE d3d11.lib d3dcompiler.lib d3dx11)
else()
    find_package(OpenGL REQUIRED)
    find_package(glfw3 3 REQUIRED)

    if (APPLE)
        target_link_libraries(imgui-node-editor PRIVATE
            "-framework CoreFoundation"
            "-framework Cocoa"
            "-framework IOKit"
            "-framework CoreVideo"
        )
    endif()
endif()

if (OpenGL_FOUND)
    set(HAVE_OPENGL YES)

    target_include_directories(imgui-node-editor PRIVATE ${OPENGL_INCLUDE_DIR})
    target_link_libraries(imgui-node-editor PRIVATE ${OPENGL_gl_LIBRARY})
    list(APPEND _Application_Sources
        src/imgui_impl_opengl3.cpp
            include/imgui-node-editor/imgui_impl_opengl3.h
            include/imgui-node-editor/imgui_impl_opengl3_loader.h
    )
endif()

if (glfw3_FOUND)
    set(HAVE_GLFW3 YES)

    list(APPEND _Application_Sources
        src/imgui_impl_glfw.cpp
            include/imgui-node-editor/imgui_impl_glfw.h
    )
    target_link_libraries(imgui-node-editor PRIVATE
        glfw
    )
endif()

configure_file(
        include/imgui-node-editor/config.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/source/config.h
)

target_compile_definitions(imgui-node-editor PRIVATE
    #BACKEND_CONFIG=IMGUI_GLFW
    #RENDERER_CONFIG=IMGUI_OGL3
)

target_include_directories(imgui-node-editor PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/source)

source_group(TREE "${CMAKE_CURRENT_SOURCE_DIR}" FILES ${_Application_Sources})

target_sources(imgui-node-editor PRIVATE ${_Application_Sources})

set_property(TARGET imgui-node-editor PROPERTY FOLDER "examples")

    file(GLOB _Example_CommonResources CONFIGURE_DEPENDS "$/examples/data/*")
    file(GLOB _Example_Resources CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/data/*")

    if (WIN32)
        set(_Example_Type WIN32)

        set(ApplicationIcon examples/Application/Support/Icon.ico)
        file(TO_NATIVE_PATH "${ApplicationIcon}" ApplicationIcon)
        string(REPLACE "\\" "\\\\" ApplicationIcon "${ApplicationIcon}")
        configure_file(
            /examples/Application/Support/Resource.rc.in
            ${CMAKE_CURRENT_BINARY_DIR}/Resource.rc
        )
        source_group(TREE "/examples" FILES ${_Example_CommonResources})
        source_group(TREE "${CMAKE_CURRENT_SOURCE_DIR}" FILES ${_Example_Resources})
        list(APPEND _Example_Resources
            ${CMAKE_CURRENT_BINARY_DIR}/Resource.rc
            ${_Example_CommonResources}
        )
        source_group("resources" FILES ${CMAKE_CURRENT_BINARY_DIR}/Resource.rc)
    elseif (APPLE)
        set(_Example_Type MACOSX_BUNDLE)

        set_source_files_properties(${_Example_Resources} ${_Example_CommonResources} PROPERTIES
            MACOSX_PACKAGE_LOCATION "Resources/data"
        )
        set(_Example_Icon "$/examples/application/support/Icon.icns")
        list(APPEND _Example_Resources ${_Example_Icon})
        set_source_files_properties(${_Example_Icon} PROPERTIES
            MACOSX_PACKAGE_LOCATION "Resources"
        )
    endif()

    add_executable(imgui-blueprint     examples/blueprints-example/blueprints-example.cpp
 ${_Example_Resources} ${_Example_CommonResources})
    
        find_package(imgui REQUIRED)
    find_package(imgui_node_editor REQUIRED)
    target_link_libraries(imgui-blueprint  PRIVATE imgui imgui_node_editor imgui-node-editor)
