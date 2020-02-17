if ( NOT DEFINED CMAKE_BUILD_TYPE )
  set( CMAKE_BUILD_TYPE Release CACHE STRING "Build type" )
endif ()

project (glew C)

cmake_minimum_required (VERSION 3.5)

include(GNUInstallDirs)

if(POLICY CMP0003)
  cmake_policy (SET CMP0003 NEW)
endif()

if(POLICY CMP0042)
  cmake_policy (SET CMP0042 NEW)
endif()

set(CMAKE_DEBUG_POSTFIX d)

option (BUILD_UTILS "utilities" ON)
option (GLEW_REGAL "Regal mode" OFF)
option (GLEW_OSMESA "OSMesa mode" OFF)
if (APPLE)
    option (BUILD_FRAMEWORK "Build Framework bundle for OSX" OFF)
endif ()

set (GLEW_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../..)

# get version from config/version
file (STRINGS ${GLEW_DIR}/config/version  _VERSION_MAJOR_STRING REGEX "GLEW_MAJOR[ ]*=[ ]*[0-9]+.*")
string (REGEX REPLACE "GLEW_MAJOR[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_MAJOR ${_VERSION_MAJOR_STRING})
file (STRINGS ${GLEW_DIR}/config/version  _VERSION_MINOR_STRING REGEX "GLEW_MINOR[ ]*=[ ]*[0-9]+.*")
string (REGEX REPLACE "GLEW_MINOR[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_MINOR ${_VERSION_MINOR_STRING})
file (STRINGS ${GLEW_DIR}/config/version  _VERSION_PATCH_STRING REGEX "GLEW_MICRO[ ]*=[ ]*[0-9]+.*")
string (REGEX REPLACE "GLEW_MICRO[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_PATCH ${_VERSION_PATCH_STRING})
set (GLEW_VERSION ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH})

set (CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set (CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set (CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

find_package (OpenGL REQUIRED)

# X11 required except for Windows and Apple OSX platforms
if (NOT WIN32 AND NOT APPLE)
  find_package (X11)
endif()

set (GLEW_LIBRARIES ${OPENGL_LIBRARIES} ${X11_LIBRARIES})

#### Regal mode ####

if (GLEW_REGAL)
  if (WIN32)
    set (REGAL_LIB_NAME regal32)
  else ()
    set (REGAL_LIB_NAME Regal)
  endif ()
  add_definitions (-DGLEW_REGAL)
  set (GLEW_LIBRARIES ${REGAL_LIB_NAME})
endif ()

#### OSMesa mode ####

if (GLEW_OSMESA)
  if (WIN32)
    set (OSMESA_LIB_NAME osmesa)
  else ()
    set (OSMESA_LIB_NAME OSMesa)
  endif ()
  add_definitions (-DGLEW_OSMESA)
  set (GLEW_LIBRARIES ${OSMESA_LIB_NAME} ${OPENGL_LIBRARIES})
  set (X11_LIBRARIES)
endif ()

#### EGL ####

if (GLEW_EGL AND UNIX)
  add_definitions (-DGLEW_EGL)
  if (OpenGL::EGL)
    message (FATAL_ERROR "EGL library set but not found.")
  endif()
  set (GLEW_LIBRARIES ${OPENGL_LIBRARIES} ${OPENGL_egl_LIBRARY})
endif ()

#### GLEW ####

include_directories (${X11_INCLUDE_DIR})

add_library(glew SHARED "")

add_library(glew_s STATIC "")

target_sources(glew
  PUBLIC
    ${GLEW_DIR}/include/GL/wglew.h
    ${GLEW_DIR}/include/GL/glew.h
    ${GLEW_DIR}/include/GL/glxew.h
  PRIVATE
    ${GLEW_DIR}/src/glew.c
    $<$<PLATFORM_ID:Windows>:${GLEW_DIR}/build/glew.rc>
)

target_sources(glew_s
  PUBLIC
    ${GLEW_DIR}/include/GL/wglew.h
    ${GLEW_DIR}/include/GL/glew.h
    ${GLEW_DIR}/include/GL/glxew.h
  PRIVATE
    ${GLEW_DIR}/src/glew.c
    $<$<PLATFORM_ID:Windows>:${GLEW_DIR}/build/glew.rc>
)

set_target_properties(glew PROPERTIES
  OUTPUT_NAME GLEW
  PREFIX lib
  VERSION ${GLEW_VERSION}
  SOVERSION ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}
  FRAMEWORK_VERSION ${GLEW_VERSION}
  MACOSX_FRAMEWORK_IDENTIFIER net.sourceforge.glew
  MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${GLEW_VERSION}
  MACOSX_FRAMEWORK_BUNDLE_VERSION ${GLEW_VERSION}
  XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
)

set_target_properties(glew_s PROPERTIES
  OUTPUT_NAME GLEW
  PREFIX lib
)

if(WIN32)
  set_target_properties(glew PROPERTIES
    OUTPUT_NAME glew32
    PREFIX ""
  )

  set_target_properties(glew_s PROPERTIES
    OUTPUT_NAME glew32
    PREFIX ""
  )
endif()

if(BUILD_FRAMEWORK)
  set_target_properties(glew PROPERTIES FRAMEWORK TRUE)
endif()

target_compile_definitions(glew
  PUBLIC
    GLEW_BUILD
    GLEW_NO_GLU
  PRIVATE
    $<$<CXX_COMPILER_ID:MSVC>:VC_EXTRALEAN>
)

target_compile_definitions(glew_s
  PUBLIC
    GLEW_STATIC
    GLEW_NO_GLU
  PRIVATE
    $<$<CXX_COMPILER_ID:MSVC>:VC_EXTRALEAN>
)

target_compile_options(glew
  PRIVATE
    # Remove security checks which are dependent on stdlib
    $<$<CXX_COMPILER_ID:MSVC>:-GS->

    # Remove builtin function dependency on Windows with GCC and Clang compiler
    # to allow usage of the library with any compiler.
    $<$<AND,$<PLATFORM_ID:Windows>,$<CXX_COMPILER_ID:GNU>>:-fno-builtin>
    $<$<AND,$<PLATFORM_ID:Windows>,$<CXX_COMPILER_ID:Clang>>:-fno-builtin>
)

target_compile_options(glew_s
  PRIVATE
    # Remove security checks which are dependent on stdlib
    $<$<CXX_COMPILER_ID:MSVC>:-GS->

    # Remove builtin functions dependency on Windows with GCC and Clang compiler.
    $<$<AND,$<PLATFORM_ID:Windows>,$<CXX_COMPILER_ID:GNU>>:-fno-builtin>
    $<$<AND,$<PLATFORM_ID:Windows>,$<CXX_COMPILER_ID:Clang>>:-fno-builtin>

    # Remove stack protection (which uses builtin functions) on Windows with GCC
    # and Clang compiler.
    $<$<AND,$<PLATFORM_ID:Windows>,$<CXX_COMPILER_ID:GNU>>:-fno-stack-protector>
    $<$<AND,$<PLATFORM_ID:Windows>,$<CXX_COMPILER_ID:Clang>>:-fno-stack-protector>
)

target_link_libraries(glew
  PUBLIC
    ${GLEW_LIBRARIES}
  PRIVATE
    # Add options from visual studio project.
    $<$<CXX_COMPILER_ID:MSVC>:-BASE:0x62AA0000>

    # Remove stdlib dependency for MSVC compiler.
    $<$<CXX_COMPILER_ID:MSVC>:-nodefaultlib>
    $<$<CXX_COMPILER_ID:MSVC>:-noentry>
    $<$<CXX_COMPILER_ID:MSVC>:libvcruntime.lib>
    $<$<CXX_COMPILER_ID:MSVC>:msvcrt.lib>

    # Remove stdlib dependency on Windows with GCC and Clang compiler.
    $<$<AND,$<PLATFORM_ID:Windows>,$<CXX_COMPILER_ID:GNU>>:-nostdlib>
    $<$<AND,$<PLATFORM_ID:Windows>,$<CXX_COMPILER_ID:Clang>>:-nostdlib>
)

target_link_libraries(glew_s
  PUBLIC
    ${GLEW_LIBRARIES}
)

if (MSVC)
  string(REGEX REPLACE "/RTC(su|[1su])" "" CMAKE_C_FLAGS_DEBUG ${CMAKE_C_FLAGS_DEBUG})
endif ()

foreach(t glew glew_s)
  target_include_directories(${t} PUBLIC $<INSTALL_INTERFACE:include>)
endforeach()

set(targets_to_install "")
if(NOT DEFINED BUILD_SHARED_LIBS OR BUILD_SHARED_LIBS)
  list(APPEND targets_to_install glew)
endif()

if(NOT DEFINED BUILD_SHARED_LIBS OR NOT BUILD_SHARED_LIBS)
  list(APPEND targets_to_install glew_s)
endif()

install ( TARGETS ${targets_to_install}
          EXPORT glew-targets
          RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
          LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
          ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
          FRAMEWORK DESTINATION ${CMAKE_INSTALL_PREFIX}
)

if (BUILD_UTILS)
  add_executable(glewinfo
    ${GLEW_DIR}/src/glewinfo.c
    $<$<PLATFORM_ID:Windows>:${GLEW_DIR}/build/glewinfo.rc>
  )

  target_link_libraries(glewinfo
    glew
    $<$<NOT,$<PLATFORM_ID:Windows>>:${X11_LIBRARIES}>
  )

  add_executable(visualinfo
    ${GLEW_DIR}/src/visualinfo.c
    $<$<PLATFORM_ID:Windows>:${GLEW_DIR}/build/visualinfo.rc>
  )

  target_link_libraries(visualinfo
    glew
    $<$<NOT,$<PLATFORM_ID:Windows>>:${X11_LIBRARIES}>
  )

  install ( TARGETS glewinfo visualinfo
            DESTINATION ${CMAKE_INSTALL_BINDIR})
endif ()

set (prefix ${CMAKE_INSTALL_PREFIX})
set (exec_prefix ${CMAKE_INSTALL_PREFIX})
set (libdir ${CMAKE_INSTALL_FULL_LIBDIR})
set (includedir ${CMAKE_INSTALL_FULL_INCLUDEDIR})
set (version ${GLEW_VERSION})
set (libname ${GLEW_LIB_NAME})
set (cflags)
set (requireslib glu)

#  Mac OSX has no glu.pc unless optional X11/GLX is installed
if (APPLE)
  set (requireslib)
endif ()

configure_file (${GLEW_DIR}/glew.pc.in ${GLEW_DIR}/glew.pc @ONLY)

install(FILES ${GLEW_DIR}/glew.pc
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)

if(WIN32 AND MSVC AND (NOT MSVC_VERSION LESS 1600) AND (NOT CMAKE_VERSION VERSION_LESS "3.1"))
    install(
        FILES $<TARGET_PDB_FILE:glew>
        DESTINATION ${CMAKE_INSTALL_LIBDIR}
        CONFIGURATIONS Debug RelWithDebInfo
    )
endif()

install (FILES
    ${GLEW_DIR}/include/GL/wglew.h
    ${GLEW_DIR}/include/GL/glew.h
    ${GLEW_DIR}/include/GL/glxew.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/GL)

install(EXPORT glew-targets DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/glew
  NAMESPACE GLEW::)
install(FILES
  ${CMAKE_CURRENT_SOURCE_DIR}/glew-config.cmake
  ${CMAKE_CURRENT_SOURCE_DIR}/CopyImportedTargetProperties.cmake
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/glew)

if(NOT TARGET uninstall)
  configure_file(
      ${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in
      ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake
      IMMEDIATE @ONLY)

  add_custom_target(uninstall
      COMMAND ${CMAKE_COMMAND} -P
      ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)
endif()
