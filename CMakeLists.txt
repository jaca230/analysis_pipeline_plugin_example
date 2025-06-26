cmake_minimum_required(VERSION 3.14)

project(example_pipeline_plugin VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

include(GNUInstallDirs)

find_package(spdlog REQUIRED)
find_package(nlohmann_json REQUIRED)
find_package(AnalysisPipelineStagesCore REQUIRED)
find_package(ROOT REQUIRED COMPONENTS Core RIO)
include(${ROOT_USE_FILE})

# Gather source files
file(GLOB_RECURSE SOURCES CONFIGURE_DEPENDS src/*.cpp)

# Create shared library
add_library(${PROJECT_NAME} SHARED ${SOURCES})

# Setup include directories
target_include_directories(${PROJECT_NAME}
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
        $<TARGET_PROPERTY:AnalysisPipelineStagesCore::analysis_pipeline_stages_core,INTERFACE_INCLUDE_DIRECTORIES>
)

# Link only against core and ROOT needed for dictionary generation
target_link_libraries(${PROJECT_NAME}
    PUBLIC
        AnalysisPipelineStagesCore::analysis_pipeline_stages_core
        ROOT::Core
        ROOT::RIO
)

# ROOT dictionary generation setup
set(ROOT_LINKDEF ${CMAKE_CURRENT_SOURCE_DIR}/include/LinkDef.h)

# Get the target's include directories directly (not using generator expressions)
get_target_property(CORE_INCLUDE_DIRS AnalysisPipelineStagesCore::analysis_pipeline_stages_core INTERFACE_INCLUDE_DIRECTORIES)

# Handle case where CORE_INCLUDE_DIRS might be CORE_INCLUDE_DIRS-NOTFOUND
if(CORE_INCLUDE_DIRS STREQUAL "CORE_INCLUDE_DIRS-NOTFOUND")
    set(CORE_INCLUDE_DIRS "")
endif()

set(ALL_INCLUDE_DIRS
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CORE_INCLUDE_DIRS}
)

# Remove duplicates and empty entries
list(REMOVE_DUPLICATES ALL_INCLUDE_DIRS)
list(REMOVE_ITEM ALL_INCLUDE_DIRS "")

message(STATUS "All include directories for ROOT dictionary: ${ALL_INCLUDE_DIRS}")

set(ROOT_DICTIONARY_INCLUDE_OPTIONS "")
foreach(dir IN LISTS ALL_INCLUDE_DIRS)
    if(EXISTS "${dir}")
        list(APPEND ROOT_DICTIONARY_INCLUDE_OPTIONS "-I${dir}")
    else()
        message(WARNING "Include directory does not exist: ${dir}")
    endif()
endforeach()

message(STATUS "ROOT dictionary include options: ${ROOT_DICTIONARY_INCLUDE_OPTIONS}")

root_generate_dictionary(
    G__${PROJECT_NAME}
    "stages/examples/constant_value_stage.h"
    MODULE ${PROJECT_NAME}
    LINKDEF ${ROOT_LINKDEF}
    OPTIONS ${ROOT_DICTIONARY_INCLUDE_OPTIONS}
)

target_link_libraries(${PROJECT_NAME} PRIVATE G__${PROJECT_NAME})

target_include_directories(${PROJECT_NAME}
    PRIVATE ${CMAKE_CURRENT_BINARY_DIR}  # generated headers
)

# Install targets and headers
install(TARGETS ${PROJECT_NAME}
    EXPORT ${PROJECT_NAME}Targets
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

install(DIRECTORY include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

include(CMakePackageConfigHelpers)

write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/Config.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
    INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
)

install(EXPORT ${PROJECT_NAME}Targets
    FILE ${PROJECT_NAME}Targets.cmake
    NAMESPACE ${PROJECT_NAME}::
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
)

install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
)