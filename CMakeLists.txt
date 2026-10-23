# =============================================================================
# How to build (run these in a terminal, NOT in this file):
#
#   cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug   # 1. configure: generates build/
#   cmake --build build -j                         # 2. compile (-j = use all CPU cores)
#   ./build/renderer_pbr                              # 3. run
#
# -S = where the sources are (here: current folder "."),
# -B = where generated files go (here: "build/"). Delete build/ to start fresh.
# =============================================================================

# Oldest CMake version allowed to process this file.
# 3.24 is needed for the "glslc" component of find_package(Vulkan).
cmake_minimum_required(VERSION 3.24)

# Project name, version, and languages used (CXX = C++).
project(rendererPBR VERSION 0.1 LANGUAGES CXX)

# Generate build/compile_commands.json, which tells your editor
# (clangd / IntelliSense) the exact flags and include paths of each file.
# Must be set BEFORE add_executable(), otherwise it has no effect on it.
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# -----------------------------------------------------------------------------
# The executable "target"
# -----------------------------------------------------------------------------
# A target is something CMake builds (here: a program named "renderer_pbr").
# Every target_xxx() command below configures this target.
# When you add new .cpp files, list them here.
add_executable(renderer_pbr src/main.cpp)

# Compile this target in C++20.
# PRIVATE = applies to renderer_pbr only (always use PRIVATE for an executable).
target_compile_features(renderer_pbr PRIVATE cxx_std_20)

# Enable compiler warnings. The $<...> syntax is a "generator expression":
# it means "add -Wall -Wextra -Wpedantic only if the compiler is GCC or Clang,
# and /W4 only if it is MSVC (Visual Studio)".
target_compile_options(renderer_pbr PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall -Wextra -Wpedantic>
    $<$<CXX_COMPILER_ID:MSVC>:/W4>)

# -----------------------------------------------------------------------------
# Libraries already installed on the system: find_package()
# -----------------------------------------------------------------------------
# Look for the Vulkan SDK (version 1.3 or newer). REQUIRED = stop with an error
# if not found. COMPONENTS glslc = also find the shader compiler, which becomes
# available as "Vulkan::glslc".
find_package(Vulkan 1.3 REQUIRED COMPONENTS glslc)

# Look for GLFW (window + input library), version 3.3 or newer.
find_package(glfw3 3.3 REQUIRED)

# Link the libraries to our program. "Vulkan::Vulkan" and "glfw" are targets
# created by find_package(): linking them also adds their include folders
# automatically, so no paths to write by hand.
target_link_libraries(renderer_pbr PRIVATE Vulkan::Vulkan glfw)


# -----------------------------------------------------------------------------
# Libraries downloaded automatically: FetchContent
# -----------------------------------------------------------------------------
# Built-in CMake module that downloads a dependency during configuration
# (step 1). Useful for libraries that are not installed on the system.
include(FetchContent)

# Describe where to get GLM (math library: vec3, mat4, ...).
FetchContent_Declare(glm
    GIT_REPOSITORY https://github.com/g-truc/glm.git
    GIT_TAG        1.0.1        # always pin an exact version (tag or commit)
    GIT_SHALLOW    TRUE)        # download only that version, not the whole history

# Actually download it and make its targets (here: glm::glm) available.
FetchContent_MakeAvailable(glm)

target_link_libraries(renderer_pbr PRIVATE glm::glm)


# -----------------------------------------------------------------------------
# Shaders: compile GLSL (.vert/.frag/.comp) into SPIR-V (.spv) for Vulkan
# -----------------------------------------------------------------------------
# Collect every shader file found in shaders/ (and its subfolders) into the
# variable SHADER_SOURCES. CONFIGURE_DEPENDS = re-scan the folder at each build,
# so newly added shader files are picked up automatically.
# ${CMAKE_SOURCE_DIR} = the project root (the folder containing this file).
file(GLOB_RECURSE SHADER_SOURCES CONFIGURE_DEPENDS
    ${CMAKE_SOURCE_DIR}/shaders/*.vert
    ${CMAKE_SOURCE_DIR}/shaders/*.frag
    ${CMAKE_SOURCE_DIR}/shaders/*.comp)

# Empty list that will hold the paths of all the .spv files to produce.
set(SPIRV_FILES "")

# Loop over each shader file (SHADER = full path of the current file).
foreach(SHADER ${SHADER_SOURCES})
    # NAME = file name only, e.g. "/.../shaders/pbr.frag" -> "pbr.frag"
    get_filename_component(NAME ${SHADER} NAME)

    # Output path, e.g. build/shaders/pbr.frag.spv
    # ${CMAKE_BINARY_DIR} = the build folder (build/).
    set(SPIRV ${CMAKE_BINARY_DIR}/shaders/${NAME}.spv)

    # Describe HOW to produce the .spv file. It is only a rule: it runs only
    # when something needs ${SPIRV} and the shader changed since last build.
    add_custom_command(
        OUTPUT  ${SPIRV}                       # the file this rule produces
        COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/shaders
        COMMAND Vulkan::glslc --target-env=vulkan1.3 ${SHADER} -o ${SPIRV}
        DEPENDS ${SHADER}                      # re-run when this file changes
        COMMENT "Compiling shader ${NAME}")

    # Remember this .spv in the list.
    list(APPEND SPIRV_FILES ${SPIRV})
endforeach()

# A target named "shaders" that "needs" all the .spv files, which triggers
# the rules above. You can also build it alone: cmake --build build -t shaders
add_custom_target(shaders DEPENDS ${SPIRV_FILES})

# Make sure shaders are compiled before the program, at every build.
add_dependencies(renderer_pbr shaders)

# Pass the shader folder path to the C++ code as a #define, so you can write:
#   std::string path = std::string(SHADER_DIR) + "pbr.frag.spv";
target_compile_definitions(renderer_pbr PRIVATE SHADER_DIR="${CMAKE_BINARY_DIR}/shaders/")
