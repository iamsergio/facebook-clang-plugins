# This is the top-level CMakeLists.txt file for the Clazy project.
#
# To build the man page from POD, run 'make man' after CMake (assumes perl is available)
# To install the resulting man page, run 'make install'
# The man page is not available on Windows.
#

project(clang-ast-dumper)

cmake_minimum_required(VERSION 3.3)
include(FeatureSummary)
include(GenerateExportHeader)
include("GNUInstallDirs")


set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_LIST_DIR}/cmake)
find_package(Clang 4.0 MODULE REQUIRED)


set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

add_definitions(-D__STDC_LIMIT_MACROS -D__STDC_CONSTANT_MACROS)
add_definitions(-D_GNU_SOURCE -DHAVE_CLANG_CONFIG_H)

option(LINK_PLUGIN_TO_LLVM "Links the clazy plugin to LLVM. Switch to OFF if your clang binary has all symbols already. Might need to be OFF if your LLVM is static." ON)


set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} -Wl,-flat_namespace -Wl,-undefined -Wl,suppress")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")


set(PLUGIN_SOURCES libtooling/ASTExporter.cpp
                   #libtooling/ast_exporter_bin.cpp
                   libtooling/FileUtils.cpp
                   libtooling/SimplePluginASTAction.cpp
                   libtooling/AttrParameterVectorStream.cpp)

include_directories(${CMAKE_BINARY_DIR})
include_directories(${CLANG_INCLUDE_DIRS} ${CMAKE_CURRENT_LIST_DIR} ${CMAKE_CURRENT_LIST_DIR}/src)
link_directories("${LLVM_INSTALL_PREFIX}/lib" ${LLVM_LIBRARY_DIRS})

macro(link_to_llvm name)
  foreach(clang_lib ${CLANG_LIBS})
    target_link_libraries(${name} ${clang_lib})
  endforeach()

  foreach(llvm_lib ${LLVM_LIBS})
    ## Don't link against LLVMSupport, causes: CommandLine Error: Option 'view-background' registered more than once!
    if (NOT llvm_lib MATCHES ".*LLVMSupport.*")
        target_link_libraries(${name} ${llvm_lib})
    endif()
  endforeach()

  foreach(user_lib ${USER_LIBS})
    target_link_libraries(${name} ${user_lib})
  endforeach()

  foreach(llvm_system_lib ${LLVM_SYSTEM_LIBS})
    target_link_libraries(${name} ${llvm_system_lib})
  endforeach()
endmacro()

macro(add_clang_plugin name)
  set(srcs ${ARGN})

  add_library(${name} SHARED ${srcs})

  if (LINK_PLUGIN_TO_LLVM)
    link_to_llvm(${name})
  endif()
endmacro()


add_clang_plugin(FBAstDumperPlugin ${PLUGIN_SOURCES})
set_target_properties(FBAstDumperPlugin PROPERTIES
LINKER_LANGUAGE CXX
PREFIX ""
)

install(TARGETS FBAstDumperPlugin
RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

# rpath
set(CMAKE_SKIP_BUILD_RPATH FALSE)
set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}")
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
list(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}" isSystemDir)
if("${isSystemDir}" STREQUAL "-1")
set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}")
endif("${isSystemDir}" STREQUAL "-1")
