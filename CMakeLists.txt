cmake_minimum_required(VERSION 3.20 FATAL_ERROR)

project(nanoid VERSION 2.1.0
  DESCRIPTION "Nanoid — compact, URL-friendly unique ID generator for C++"
  LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 11)
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)

file(GLOB_RECURSE NANOID_SRC_FILES src/*.cpp)
file(GLOB_RECURSE NANOID_INC_FILES inc/*.h)

# In root-build mode the monorepo's create_project() macro exists; use it.
# In standalone/port mode, set up the target manually.
if(COMMAND create_project)
  add_subdirectory(vendor/catch2)
  create_project(nanoid static)

  if(MSVC)
    create_project(nanoid-shared)
  endif()
else()
  add_library(${PROJECT_NAME} STATIC ${NANOID_SRC_FILES})

  target_include_directories(${PROJECT_NAME} PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/inc>
    $<INSTALL_INTERFACE:include>
  )

  target_link_libraries(${PROJECT_NAME} PUBLIC Threads::Threads)

  set_property(TARGET ${PROJECT_NAME} PROPERTY POSITION_INDEPENDENT_CODE ON)

  if(MSVC)
    set_target_properties(${PROJECT_NAME} PROPERTIES
      MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>"
    )
  else()
    target_compile_options(${PROJECT_NAME} PRIVATE
      $<$<OR:$<CXX_COMPILER_ID:Clang>,$<CXX_COMPILER_ID:GNU>>:-Wall -Wextra -Wunused -Werror>
    )
  endif()
endif()

# ---- Install ----
include(CMakePackageConfigHelpers)
include(GNUInstallDirs)

install(TARGETS ${PROJECT_NAME}
  EXPORT ${PROJECT_NAME}Targets
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)
install(DIRECTORY inc/
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  FILES_MATCHING PATTERN "*.h"
)
install(EXPORT ${PROJECT_NAME}Targets
  FILE ${PROJECT_NAME}Targets.cmake
  NAMESPACE nanoid::
  DESTINATION share/${PROJECT_NAME}
)

configure_package_config_file(cmake/${PROJECT_NAME}Config.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
  INSTALL_DESTINATION share/${PROJECT_NAME}
)
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY SameMajorVersion
)
install(FILES
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
  DESTINATION share/${PROJECT_NAME}
)

# ############################################
# Unit tests (root-build only — Catch2 vendored in vendor/)
if(COMMAND create_project)
  enable_testing()
  add_executable(nanoid_tests EXCLUDE_FROM_ALL tests/unit_tests.cpp)

  target_link_libraries(nanoid_tests
    nanoid
    Catch2::Test
  )
  add_test(test_all nanoid_tests)
  set_target_properties(nanoid_tests PROPERTIES FOLDER "third-party")
endif()
