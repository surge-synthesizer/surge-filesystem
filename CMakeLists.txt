cmake_minimum_required(VERSION 3.10)
project(filesystem VERSION 0.0.0 LANGUAGES CXX)

if(${BUILD_STANDALONE_FS_TESTS})
   set(CMAKE_CXX_STANDARD 17)
endif()

function(generate_header FS_HEADER FS_NAMESPACE)
  message(STATUS "${PROJECT_NAME}: Using ${FS_NAMESPACE} from ${FS_HEADER}")
  set(INC_DIR "${CMAKE_CURRENT_BINARY_DIR}/include")
  set(HEADER_DIR "${INC_DIR}/${PROJECT_NAME}")
  set(HEADER_FILE "${HEADER_DIR}/import.h")
  configure_file(src/import.h.in "${HEADER_FILE}" @ONLY)
  target_sources(${PROJECT_NAME} INTERFACE "${HEADER_FILE}")
  target_include_directories(${PROJECT_NAME} INTERFACE "${INC_DIR}")
endfunction()

function(use_fallback_fs)
  add_library(${PROJECT_NAME}
    src/directory_iterator.cpp
    src/filesystem.cpp
    src/path.cpp
    src/recursive_directory_iterator.cpp
    src/util.h
    include/${PROJECT_NAME}/directory_entry.h
    include/${PROJECT_NAME}/directory_iterator.h
    include/${PROJECT_NAME}/filesystem.h
    include/${PROJECT_NAME}/filesystem_error.h
    include/${PROJECT_NAME}/path.h
    include/${PROJECT_NAME}/recursive_directory_iterator.h
  )
  target_include_directories(${PROJECT_NAME} PUBLIC include)

  if(${FILESYSTEM_TEST_TARGET})
    target_sources(${FILESYSTEM_TEST_TARGET} INTERFACE src/tests.cpp)
  endif()
  generate_header("\"filesystem/filesystem.h\"" "Surge::filesystem")
endfunction()

function(use_platform_fs)
  add_library(${PROJECT_NAME} INTERFACE)
  generate_header("<filesystem>" "std::filesystem")
endfunction()

set(SURGE_DEVEL_FORCE_FALLBACK_FS OFF CACHE BOOL "Force fallback std::filesystem impl (development only)")
mark_as_advanced(FORCE SURGE_DEVEL_FORCE_FALLBACK_FS)
if (${SURGE_DEVEL_FORCE_FALLBACK_FS})
  use_fallback_fs()
else()
  include(CheckCXXSourceCompiles)
  check_cxx_source_compiles("
    #include <filesystem>
    namespace fs = std::filesystem;
    int main(int, char **argv) {
      return fs::recursive_directory_iterator{fs::path{*argv}}.depth();
    }" SURGE_FILESYSTEM_STD_FS_FOUND)
  if (SURGE_FILESYSTEM_STD_FS_FOUND)
    use_platform_fs()
  else()
    use_fallback_fs()
  endif()
endif()

add_library(surge::${PROJECT_NAME} ALIAS ${PROJECT_NAME})

if(${BUILD_STANDALONE_FS_TESTS})
   set(CMAKE_CXX_STANDARD 17)
   message(STATUS "Building Standalone Tests" )
   add_executable( filesystem-tests test-utils/test_main.cpp src/tests.cpp )
   add_dependencies( filesystem-tests surge::${PROJECT_NAME} )
   target_include_directories( filesystem-tests PRIVATE include test-utils )
   target_link_libraries(filesystem-tests PUBLIC surge::${PROJECT_NAME} )

   add_custom_command(
        TARGET filesystem-tests
        POST_BUILD
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
	COMMAND cmake -E make_directory ${CMAKE_BINARY_DIR}/products
        COMMAND cmake -E copy $<TARGET_FILE:filesystem-tests> ${CMAKE_BINARY_DIR}/products
   )
endif()
