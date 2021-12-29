# Packaging for import_utils.cmake script
#
# Copyright (c) 2021 FeRD (Frank Dana)
#
# SPDX-License-Identifier: LGPL-3.0-or-later

cmake_minimum_required(VERSION 3.5 FATAL_ERROR)

set(IMPORT_UTILS_VERSION 1.0.1)
project(ImportUtils LANGUAGES NONE VERSION ${IMPORT_UTILS_VERSION})

message(STATUS
"Packaging ${PROJECT_NAME} ${PROJECT_VERSION} using CMake ${CMAKE_VERSION}")

set(CMAKECONFIG_INSTALL_DIR "lib/cmake/${PROJECT_NAME}")

include(CMakePackageConfigHelpers)

configure_package_config_file(
  Config.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
  INSTALL_DESTINATION ${CMAKECONFIG_INSTALL_DIR}
  NO_SET_AND_CHECK_MACRO
  NO_CHECK_REQUIRED_COMPONENTS_MACRO
)

write_basic_package_version_file(
  ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
  VERSION "${PROJECT_VERSION}"
  COMPATIBILITY AnyNewerVersion
)

install(FILES
  "${CMAKE_CURRENT_SOURCE_DIR}/import_utils.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
  DESTINATION "${CMAKECONFIG_INSTALL_DIR}"
)
