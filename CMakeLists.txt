# FxOgreFBX + ExecFxOgreFBX against the ogredeps Ogre SDK and the Autodesk FBX SDK.
# Alternative to the .vcxproj files; see README.txt.
cmake_minimum_required(VERSION 3.20)
project(FxOgreFBX LANGUAGES CXX)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()
set(CMAKE_CXX_STANDARD 17)

get_filename_component(OGREDEPS_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/.." ABSOLUTE)
set(FXOGRE_OGRE_VERSION "14.5.2" CACHE STRING "Ogre version folder under ogredeps")
set(OGRE_SDK "${OGREDEPS_ROOT}/ogre-${FXOGRE_OGRE_VERSION}/SDK/${CMAKE_BUILD_TYPE}"
    CACHE PATH "Installed Ogre SDK for this configuration")
find_package(OGRE REQUIRED CONFIG PATHS "${OGRE_SDK}/CMake" NO_DEFAULT_PATH)

set(FBXSDK_ROOT "C:/Program Files/Autodesk/FBX/FBX SDK/2020.3.9" CACHE PATH "Autodesk FBX SDK")
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
  set(FBXSDK_LIBDIR "${FBXSDK_ROOT}/lib/x64/debug")
else()
  set(FBXSDK_LIBDIR "${FBXSDK_ROOT}/lib/x64/release")
endif()

set(TINYXML_DIR "${CMAKE_CURRENT_SOURCE_DIR}/External/tinyxml" CACHE PATH "tinyxml2 2.1.0 sources")
if(NOT EXISTS "${TINYXML_DIR}/tinyxml2.cpp")
  message(FATAL_ERROR "tinyxml2.h/.cpp (v2.1.0) not found in ${TINYXML_DIR}")
endif()

file(GLOB FXOGRE_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/Src/*.cpp")
add_library(FxOgreFBX SHARED ${FXOGRE_SOURCES})
target_include_directories(FxOgreFBX PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/Inc"
                           PRIVATE "${FBXSDK_ROOT}/include" "${TINYXML_DIR}")
target_compile_definitions(FxOgreFBX PRIVATE WIN32 _WINDOWS _USRDLL FXOGREFBX_EXPORTS FBXSDK_NEW_API
                           _CRT_SECURE_NO_WARNINGS)
target_compile_options(FxOgreFBX PRIVATE /MP /W3)
# Static FBX SDK against the dynamic CRT (-md variants)
target_link_libraries(FxOgreFBX PRIVATE OgreMain
  "${FBXSDK_LIBDIR}/libfbxsdk-md.lib" "${FBXSDK_LIBDIR}/libxml2-md.lib" "${FBXSDK_LIBDIR}/zlib-md.lib"
  Advapi32 Wininet)

add_executable(ExecFxOgreFBX "${CMAKE_CURRENT_SOURCE_DIR}/ExecFxOgreFBX/Src/ExecFxOgreFBX.cpp")
target_include_directories(ExecFxOgreFBX PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/ExecFxOgreFBX/Inc")
target_compile_definitions(ExecFxOgreFBX PRIVATE _CRT_SECURE_NO_WARNINGS)
target_link_libraries(ExecFxOgreFBX PRIVATE FxOgreFBX)

# Runnable bin/: tool + OgreMain next to it
set(RUN_DIR "${CMAKE_BINARY_DIR}/bin")
set_target_properties(FxOgreFBX ExecFxOgreFBX PROPERTIES
  RUNTIME_OUTPUT_DIRECTORY_DEBUG "${RUN_DIR}" RUNTIME_OUTPUT_DIRECTORY_RELEASE "${RUN_DIR}"
  RUNTIME_OUTPUT_DIRECTORY_RELWITHDEBINFO "${RUN_DIR}")
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
  set(OGREMAIN_DLL "${OGRE_SDK}/bin/OgreMain_d.dll")
else()
  set(OGREMAIN_DLL "${OGRE_SDK}/bin/OgreMain.dll")
endif()
add_custom_command(TARGET ExecFxOgreFBX POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E copy_if_different "${OGREMAIN_DLL}" "${RUN_DIR}")
