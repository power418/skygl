cmake_minimum_required(VERSION 4.0)
project(skygl C CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# ===== CPM (package manager) =====
include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/CPM.cmake)

# ===== Dependencies =====

CPMAddPackage(
    NAME glfw3
    GITHUB_REPOSITORY glfw/glfw
    GIT_TAG 3.4
    OPTIONS
        "GLFW_BUILD_EXAMPLES OFF"
        "GLFW_BUILD_TESTS OFF"
        "GLFW_BUILD_DOCS OFF"
        "GLFW_INSTALL OFF"
)

CPMAddPackage(
    NAME glm
    GITHUB_REPOSITORY g-truc/glm
    GIT_TAG 1.0.1
)

# glad v0.1.36 has no CMake >= 3.5 support in root CMakeLists, fetch sources only and generate at configure time
CPMAddPackage(
    NAME glad
    GITHUB_REPOSITORY Dav1dde/glad
    GIT_TAG v0.1.36
    DOWNLOAD_ONLY YES
)

find_package(Python3 COMPONENTS Interpreter REQUIRED)
set(GLAD_OUT_DIR ${CMAKE_BINARY_DIR}/glad-generated)
execute_process(
    COMMAND ${Python3_EXECUTABLE} -m glad --profile=core --api=gl=4.5 --generator=c --out-path=${GLAD_OUT_DIR}
    WORKING_DIRECTORY ${glad_SOURCE_DIR}
    RESULT_VARIABLE GLAD_GEN_RESULT
    ERROR_VARIABLE GLAD_GEN_ERROR
)
if(NOT GLAD_GEN_RESULT EQUAL 0)
    message(FATAL_ERROR "glad generation failed:\n${GLAD_GEN_ERROR}")
endif()

add_library(glad STATIC ${GLAD_OUT_DIR}/src/glad.c)
target_include_directories(glad PUBLIC ${GLAD_OUT_DIR}/include)
target_link_libraries(glad PUBLIC ${CMAKE_DL_LIBS})
add_library(glad::glad ALIAS glad)


CPMAddPackage(
    NAME imgui
    GITHUB_REPOSITORY ocornut/imgui
    GIT_TAG v1.92.9b
)

add_library(imgui_impl STATIC
    ${imgui_SOURCE_DIR}/imgui.cpp
    ${imgui_SOURCE_DIR}/imgui_demo.cpp
    ${imgui_SOURCE_DIR}/imgui_draw.cpp
    ${imgui_SOURCE_DIR}/imgui_tables.cpp
    ${imgui_SOURCE_DIR}/imgui_widgets.cpp
    ${imgui_SOURCE_DIR}/backends/imgui_impl_glfw.cpp
    ${imgui_SOURCE_DIR}/backends/imgui_impl_opengl3.cpp
)
target_include_directories(imgui_impl PUBLIC
    ${imgui_SOURCE_DIR}
    ${imgui_SOURCE_DIR}/backends
)
target_link_libraries(imgui_impl PUBLIC glfw glad::glad)
add_library(imgui::imgui ALIAS imgui_impl)

CPMAddPackage(
    NAME stb
    GITHUB_REPOSITORY nothings/stb
    GIT_TAG master
    DOWNLOAD_ONLY YES
)
set(Stb_INCLUDE_DIR ${stb_SOURCE_DIR})

find_package(OpenGL REQUIRED)

# ===== Compile & Link =====
set(SOURCE src/main.cpp
        src/Utils.cpp
        src/Input.cpp
        src/Sky/Atmosphere/AtmosphereModel.cpp
        src/Sky/Atmosphere/TextureBuffer.cpp
        src/Sky/Clouds/CloudsModel.cpp
        src/Gl/GlUtils.cpp
        src/Gl/Shader.cpp
        src/Gl/Texture.cpp
        src/Gl/UniformBuffer.cpp
        src/Noise.cpp
        src/Sky/SkySystem.cpp
 "src/Sky/Weather.cpp")

if (WIN32)
    list(APPEND SOURCES res/resource.rc)
endif ()

add_executable(skygl ${SOURCE})
target_include_directories(skygl PRIVATE src)
target_include_directories(skygl PRIVATE ${Stb_INCLUDE_DIR})
target_link_libraries(skygl PRIVATE OpenGL::GL glm::glm glfw glad::glad imgui::imgui)

file(CREATE_LINK "${CMAKE_CURRENT_SOURCE_DIR}/shaders" "${CMAKE_CURRENT_BINARY_DIR}/shaders" SYMBOLIC COPY_ON_ERROR)
file(CREATE_LINK "${CMAKE_CURRENT_SOURCE_DIR}/res" "${CMAKE_CURRENT_BINARY_DIR}/res" SYMBOLIC COPY_ON_ERROR)

