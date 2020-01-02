# Copyright (c) 2013-2014, Ruslan Baratov
# Copyright (c) 2019-2020 Cristian Adam
# All rights reserved.

cmake_minimum_required(VERSION 3.2)

if (NOT HUNTER_URL AND NOT HUNTER_SHA1)
  file(
    DOWNLOAD https://api.github.com/repos/cpp-pm/hunter/releases/latest
    ${CMAKE_BINARY_DIR}/LatestGate.json)

  file(READ ${CMAKE_BINARY_DIR}/LatestGate.json latestGateJson)
  string(REGEX MATCH "tag_name\": \"([a-z0-9.]+)\"" tagName "${latestGateJson}")

  set(hunterUrlTarball "https://github.com/cpp-pm/hunter/archive/${CMAKE_MATCH_1}.tar.gz")
  set(hunterLocalTarballName "${CMAKE_BINARY_DIR}/hunter-${CMAKE_MATCH_1}.tar.gz")

  file(DOWNLOAD ${hunterUrlTarball} "${hunterLocalTarballName}")

  file(SHA1 "${hunterLocalTarballName}" HUNTER_SHA1)
  set(HUNTER_URL "file://${hunterLocalTarballName}")
endif()

if (NOT HUNTER_LOCAL_CONFIG OR
    NOT HUNTER_GLOBAL_CONFIG OR
    NOT HUNTER_FILEPATH_CONFIG OR
    HUNTER_PACKAGES)
  # Write the Hunter Configuration file
  file(WRITE ${CMAKE_BINARY_DIR}/HunterConfig.cmake.in "")
  foreach(package IN LISTS HUNTER_PACKAGES)
    file(APPEND ${CMAKE_BINARY_DIR}/HunterConfig.cmake.in
      "hunter_config(${package}
        VERSION ")
    if (HUNTER_${package}_VERSION)
      file(APPEND ${CMAKE_BINARY_DIR}/HunterConfig.cmake.in
        "${HUNTER_${package}_VERSION}")
    else()
      file(APPEND ${CMAKE_BINARY_DIR}/HunterConfig.cmake.in
        "\${HUNTER_${package}_VERSION}")
    endif()

    if (HUNTER_${package}_CMAKE_ARGS)
      file(APPEND ${CMAKE_BINARY_DIR}/HunterConfig.cmake.in "
        CMAKE_ARGS \${HUNTER_${package}_CMAKE_ARGS}")
    endif()

    file(APPEND ${CMAKE_BINARY_DIR}/HunterConfig.cmake.in ")\n\n")
  endforeach()
  configure_file(
    ${CMAKE_BINARY_DIR}/HunterConfig.cmake.in
    ${CMAKE_BINARY_DIR}/HunterConfig.cmake @ONLY)
endif()

unset(HUNTER_GATE_CONFIG)
if (HUNTER_LOCAL_CONFIG)
  set(HUNTER_GATE_CONFIG LOCAL)
elseif (HUNTER_GLOBAL_CONFIG)
  set(HUNTER_GATE_CONFIG GLOBAL ${HUNTER_GLOBAL_CONFIG})
elseif (HUNTER_FILEPATH_CONFIG)
  set(HUNTER_GATE_CONFIG FILEPATH ${HUNTER_FILEPATH_CONFIG})
elseif (HUNTER_PACKAGES)
  set(HUNTER_GATE_CONFIG FILEPATH ${CMAKE_BINARY_DIR}/HunterConfig.cmake)
endif()

file(WRITE
  ${CMAKE_BINARY_DIR}/HunterSetup.cmake.in
  "include(\"${CMAKE_CURRENT_LIST_DIR}/cmake/HunterGate.cmake\")\n"
  "HunterGate(
    URL ${HUNTER_URL}
    SHA1 ${HUNTER_SHA1}
    ${HUNTER_GATE_CONFIG}
   )\n"
)

configure_file(
  ${CMAKE_BINARY_DIR}/HunterSetup.cmake.in
  ${CMAKE_BINARY_DIR}/HunterSetup.cmake @ONLY)

# Build only the current CMAKE_BUILD_TYPE
if (NOT HUNTER_CONFIGURATION_TYPES AND CMAKE_BUILD_TYPE)
  set(HUNTER_CONFIGURATION_TYPES ${CMAKE_BUILD_TYPE} CACHE STRING "Configurations HUNTER will build")
endif()

include(${CMAKE_BINARY_DIR}/HunterSetup.cmake)

#
# Create a toolchain file
#

file(WRITE ${CMAKE_BINARY_DIR}/HunterToolchain.cmake.in
[=[
# Add hunter install directory to the find_package variables
if (EXISTS "${CMAKE_CURRENT_LIST_DIR}/_3rdParty/Hunter/install-root-dir")
  file(READ "${CMAKE_CURRENT_LIST_DIR}/_3rdParty/Hunter/install-root-dir" HunterInstall)
  list(APPEND CMAKE_FIND_ROOT_PATH "${HunterInstall}")
  list(APPEND CMAKE_PREFIX_PATH "${HunterInstall}")
endif()
]=])

if(ANDROID_ABI AND ANDROID_NATIVE_API_LEVEL)
  file(APPEND ${CMAKE_BINARY_DIR}/HunterToolchain.cmake.in
    "set(ANDROID_ABI ${ANDROID_ABI})\n"
    "set(ANDROID_NATIVE_API_LEVEL ${ANDROID_NATIVE_API_LEVEL})\n"
    # Android has RPATH, and we dont' have to relink when installed\n"
    "set(CMAKE_BUILD_WITH_INSTALL_RPATH ON)\n")
endif()

foreach(lang IN ITEMS C CXX)
  if(CMAKE_${lang}_COMPILER_LAUNCHER)
    file(APPEND ${CMAKE_BINARY_DIR}/HunterToolchain.cmake.in
      "set(CMAKE_${lang}_COMPILER_LAUNCHER ${CMAKE_${lang}_COMPILER_LAUNCHER})\n")
  endif()
endforeach()

if (CMAKE_TOOLCHAIN_FILE)
  file(APPEND ${CMAKE_BINARY_DIR}/HunterToolchain.cmake.in
    "include(\"${CMAKE_TOOLCHAIN_FILE}\")\n")
endif()

configure_file(
  ${CMAKE_BINARY_DIR}/HunterToolchain.cmake.in
  ${CMAKE_BINARY_DIR}/HunterToolchain.cmake @ONLY)

# Setup a project, and add the packages
project(HunterSetup)

#
# Building the packages
#

# Build the hunter packages with the configured toolchain
if (CMAKE_TOOLCHAIN_FILE)
  set(ORIGINAL_TOOLCHAIN_FILE ${CMAKE_TOOLCHAIN_FILE})
  set(CMAKE_TOOLCHAIN_FILE ${CMAKE_BINARY_DIR}/HunterToolchain.cmake)
endif()

foreach(package IN LISTS HUNTER_PACKAGES)
  unset(_package_components)
  if (HUNTER_${package}_COMPONENTS)
    set(_package_components COMPONENTS ${HUNTER_${package}_COMPONENTS})
  endif()
  hunter_add_package(${package} ${_package_components})
endforeach()

# Restore the original TOOLCHAIN_FILE
if (ORIGINAL_TOOLCHAIN_FILE)
  set(CMAKE_TOOLCHAIN_FILE "${ORIGINAL_TOOLCHAIN_FILE}")
endif()

#
# Make sure the parent find_package and include calls work as expected
#

set(CMAKE_FIND_ROOT_PATH "${CMAKE_FIND_ROOT_PATH}" PARENT_SCOPE)
set(CMAKE_PREFIX_PATH "${CMAKE_PREFIX_PATH}" PARENT_SCOPE)
set(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH}" PARENT_SCOPE)

#
# Variables expected to be found in global scope
#
set(HUNTER_ID_PATH "${HUNTER_ID_PATH}" PARENT_SCOPE)
set(HUNTER_TOOLCHAIN_ID_PATH "${HUNTER_TOOLCHAIN_ID_PATH}" PARENT_SCOPE)
set(HUNTER_CONFIG_ID_PATH "${HUNTER_CONFIG_ID_PATH}" PARENT_SCOPE)
