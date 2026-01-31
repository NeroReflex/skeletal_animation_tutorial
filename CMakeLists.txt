cmake_minimum_required(VERSION 3.10)

project(SkeletalAnimationDemo LANGUAGES CXX)

option(USE_SYSTEM_LIBS "Prefer system-installed libraries (GLFW, Assimp, GLM)" ON)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Recommended compiler warnings
if (CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# Allow GLM experimental extensions used by the code
add_compile_definitions(GLM_ENABLE_EXPERIMENTAL)

# Source files (collect all .cpp in TeapotAD)
file(GLOB TEAPOT_SOURCES
  "TeapotAD/*.cpp"
)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/TeapotAD)

# GLM: prefer system, otherwise fall back to bundled copy in Libs/glm
if (USE_SYSTEM_LIBS)
  find_package(glm QUIET)
endif()
if (NOT glm_FOUND)
  message(STATUS "Using bundled GLM in Libs/glm")
  include_directories(${CMAKE_CURRENT_SOURCE_DIR}/Libs/glm)
else()
  message(STATUS "Found system GLM")
  include_directories(${GLM_INCLUDE_DIRS})
endif()

# stb_image: use bundled headers if present
if (EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/Libs/stb_image/stb_image")
  include_directories(${CMAKE_CURRENT_SOURCE_DIR}/Libs/stb_image)
endif()

# gl load: if project already provides gl_core in TeapotAD, include current directory
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/TeapotAD)

# Find OpenGL and GLFW
find_package(OpenGL REQUIRED)

if (USE_SYSTEM_LIBS)
  find_package(PkgConfig QUIET)
  if (PkgConfig_FOUND)
    pkg_check_modules(GLFW_PKG QUIET glfw3)
  endif()
  if (GLFW_PKG_FOUND)
    message(STATUS "Found GLFW via pkg-config")
    include_directories(${GLFW_PKG_INCLUDEDIR})
    set(GLFW_LIBRARIES ${GLFW_PKG_LIBRARIES})
  else()
    find_package(glfw3 QUIET)
  endif()
endif()

if (NOT TARGET glfw AND NOT GLFW_LIBRARIES)
  # try to link libglfw (system) by name
  find_library(GLFW_LIBRARY NAMES glfw glfw3)
  if (GLFW_LIBRARY)
    set(GLFW_LIBRARIES ${GLFW_LIBRARY})
  endif()
endif()

# Assimp: prefer system
if (USE_SYSTEM_LIBS)
  find_package(assimp QUIET)
endif()
if (assimp_FOUND)
  message(STATUS "Found system Assimp")
  include_directories(${ASSIMP_INCLUDE_DIR})
else()
  # try bundled include if present
  if (EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/Libs/Assimp/include")
    message(STATUS "Using bundled Assimp headers (no bundled library) — you may need to install Assimp dev package")
    include_directories(${CMAKE_CURRENT_SOURCE_DIR}/Libs/Assimp/include)
  endif()
endif()

# Create target
add_executable(TeapotAD ${TEAPOT_SOURCES})

# Include directories specific to project
target_include_directories(TeapotAD PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/TeapotAD)

# Link libraries: OpenGL, GLFW, Assimp (if found), pthread and dl
if (TARGET OpenGL::GL)
  target_link_libraries(TeapotAD PRIVATE OpenGL::GL)
else()
  target_link_libraries(TeapotAD PRIVATE ${OPENGL_LIBRARIES})
endif()

if (GLFW_LIBRARIES)
  target_link_libraries(TeapotAD PRIVATE ${GLFW_LIBRARIES})
endif()

if (assimp_FOUND)
  target_link_libraries(TeapotAD PRIVATE assimp::assimp)
else()
  # attempt to link generic assimp library name
  find_library(ASSIMP_LIB NAMES assimp)
  if (ASSIMP_LIB)
    target_link_libraries(TeapotAD PRIVATE ${ASSIMP_LIB})
  endif()
endif()

find_package(Threads QUIET)
if (THREADS_FOUND)
  target_link_libraries(TeapotAD PRIVATE Threads::Threads)
endif()

find_library(DL_LIB dl)
if (DL_LIB)
  target_link_libraries(TeapotAD PRIVATE ${DL_LIB})
endif()

# If glfw target exists, link it
if (TARGET glfw)
  target_link_libraries(TeapotAD PRIVATE glfw)
endif()

# Set default build type
if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the build type." FORCE)
endif()

message(STATUS "Configuration complete. Use: mkdir build && cd build && cmake .. && make")
