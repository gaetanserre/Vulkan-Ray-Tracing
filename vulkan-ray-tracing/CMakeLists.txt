#*****************************************************************************
# Copyright 2020 NVIDIA Corporation. All rights reserved.
#*****************************************************************************

cmake_minimum_required(VERSION 3.31 FATAL_ERROR)

project(vulkan_ray_tracing)


#--------------------------------------------------------------------------------------------------
# Check if nvpro_core is already cloned
if(NOT EXISTS ${CMAKE_SOURCE_DIR}/../nvpro_core)
  message(STATUS "nvpro_core not found, cloning...")
  execute_process(
    COMMAND git clone --recursive --shallow-submodules git@github.com:gaetanserre/nvpro_core.git
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/..
  )

else()
  message(STATUS "nvpro_core already exists, skipping clone.")
endif()

#--------------------------------------------------------------------------------------------------
# look for nvpro_core 1) as a sub-folder 2) at some other locations
# this cannot be put anywhere else since we still didn't find setup.cmake yet
if(NOT BASE_DIRECTORY)

  find_path(BASE_DIRECTORY
    NAMES nvpro_core/cmake/setup.cmake
    PATHS ${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/.. ${CMAKE_CURRENT_SOURCE_DIR}/../.. 
    REQUIRED
    DOC "Directory containing nvpro_core"
    )
endif()

## Various functions and macros REQUIRED
if(EXISTS ${BASE_DIRECTORY}/nvpro_core/cmake/setup.cmake)
  include(${BASE_DIRECTORY}/nvpro_core/cmake/setup.cmake)
  include(${BASE_DIRECTORY}/nvpro_core/cmake/utilities.cmake)
else()
  message(FATAL_ERROR "could not find base directory, please set BASE_DIRECTORY to folder containing nvpro_core")
endif()

#--------------------------------------------------------------------------------------------------
# Package shared by all projects
_add_package_VulkanSDK()
_add_package_ImGUI()
_add_nvpro_core_lib()

message(STATUS "COPY ${CMAKE_CURRENT_SOURCE_DIR}/media to ${CMAKE_CURRENT_BINARY_DIR}")
file(COPY ${CMAKE_CURRENT_SOURCE_DIR}/media DESTINATION ${CMAKE_CURRENT_BINARY_DIR})


#--------------------------------------------------------------------------------------------------
# C++ target and defines
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3")
add_executable(vulkan_ray_tracing)
_add_project_definitions(vulkan_ray_tracing)


#--------------------------------------------------------------------------------------------------
# Source files for this project
#
file(GLOB SOURCE_FILES src/*.cpp *.hpp *.inl *.c)
file(GLOB EXTRA_COMMON ${CMAKE_SOURCE_DIR}/obj_loader/*.*)
list(APPEND COMMON_SOURCE_FILES ${EXTRA_COMMON})
include_directories(${CMAKE_SOURCE_DIR}/obj_loader ${CMAKE_SOURCE_DIR}/include)


#--------------------------------------------------------------------------------------------------
# GLSL to SPIR-V custom build
compile_glsl_directory(
	SRC "${CMAKE_CURRENT_SOURCE_DIR}/shaders" 
	DST "${CMAKE_CURRENT_BINARY_DIR}/spv" 
	VULKAN_TARGET "vulkan1.4"
	DEPENDENCY ${VULKAN_BUILD_DEPENDENCIES}
	)


#--------------------------------------------------------------------------------------------------
# Sources
target_sources(vulkan_ray_tracing PUBLIC ${SOURCE_FILES} ${HEADER_FILES})
target_sources(vulkan_ray_tracing PUBLIC ${COMMON_SOURCE_FILES})
target_sources(vulkan_ray_tracing PUBLIC ${PACKAGE_SOURCE_FILES})
target_sources(vulkan_ray_tracing PUBLIC ${GLSL_SOURCES} ${GLSL_HEADERS})


#--------------------------------------------------------------------------------------------------
# Sub-folders in Visual Studio
#



#--------------------------------------------------------------------------------------------------
# Linkage
#
target_link_libraries(vulkan_ray_tracing ${PLATFORM_LIBRARIES} nvpro_core)

foreach(DEBUGLIB ${LIBRARIES_DEBUG})
  target_link_libraries(vulkan_ray_tracing debug ${DEBUGLIB})
endforeach(DEBUGLIB)

foreach(RELEASELIB ${LIBRARIES_OPTIMIZED})
  target_link_libraries(vulkan_ray_tracing optimized ${RELEASELIB})
endforeach(RELEASELIB)

#--------------------------------------------------------------------------------------------------
# copies binaries that need to be put next to the exe files (ZLib, etc.)
#
#_finalize_target( vulkan_ray_tracing )


#install(FILES ${SPV_OUTPUT} CONFIGURATIONS Release DESTINATION "${CMAKE_CURRENT_BINARY_DIR}/spv")
#install(FILES ${SPV_OUTPUT} CONFIGURATIONS Debug DESTINATION "bin_${ARCH}_debug/vulkan_ray_tracing/spv")
