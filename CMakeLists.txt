cmake_minimum_required(VERSION 3.16)
project(glew LANGUAGES C)

if ( NOT DEFINED CMAKE_BUILD_TYPE )
  set( CMAKE_BUILD_TYPE Release CACHE STRING "Build type" )
endif ()

include(GNUInstallDirs)

set(CMAKE_DEBUG_POSTFIX d)

option (BUILD_UTILS "utilities" ON)
option (GLEW_REGAL "Regal mode" OFF)
option (GLEW_OSMESA "OSMesa mode" OFF)
if (APPLE)
    option (BUILD_FRAMEWORK "Build Framework bundle for OSX" OFF)
endif ()
option (GLEW_X11 "X11 mode" ON)
option (GLEW_EGL "EGL mode" OFF)
option (BUILD_SHARED_LIBS "install/link shared instead of static libs" ON)

set (GLEW_DIR ${CMAKE_CURRENT_SOURCE_DIR})

# get version from config/version
file (STRINGS ${GLEW_DIR}/config/version  _VERSION_MAJOR_STRING REGEX "GLEW_MAJOR[ ]*=[ ]*[0-9]+.*")
string (REGEX REPLACE "GLEW_MAJOR[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_MAJOR ${_VERSION_MAJOR_STRING})
file (STRINGS ${GLEW_DIR}/config/version  _VERSION_MINOR_STRING REGEX "GLEW_MINOR[ ]*=[ ]*[0-9]+.*")
string (REGEX REPLACE "GLEW_MINOR[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_MINOR ${_VERSION_MINOR_STRING})
file (STRINGS ${GLEW_DIR}/config/version  _VERSION_PATCH_STRING REGEX "GLEW_MICRO[ ]*=[ ]*[0-9]+.*")
string (REGEX REPLACE "GLEW_MICRO[ ]*=[ ]*([0-9]+)" "\\1" CPACK_PACKAGE_VERSION_PATCH ${_VERSION_PATCH_STRING})
set (GLEW_VERSION ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH})

find_package (OpenGL REQUIRED)

# prefer GLVND
if (OPENGL_opengl_LIBRARY)
  set (GLEW_LIBRARIES ${OPENGL_opengl_LIBRARY})
else ()
  set (GLEW_LIBRARIES ${OPENGL_gl_LIBRARY})
endif ()

# X11 required except for Windows and Apple OSX platforms
if (GLEW_X11 AND NOT WIN32 AND NOT APPLE)
  find_package (X11)
  list (APPEND GLEW_LIBRARIES ${OPENGL_glx_LIBRARY} ${X11_LIBRARIES})
endif()

if (WIN32)
  set (GLEW_LIB_NAME glew32)
else ()
  set (GLEW_LIB_NAME GLEW)
  set (DLL_PREFIX lib)
endif ()

add_definitions (-DGLEW_NO_GLU)

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
  list (APPEND GLEW_LIBRARIES ${OSMESA_LIB_NAME})
  set (X11_LIBRARIES)
endif ()

#### EGL ####

if (GLEW_EGL AND UNIX)
  add_definitions (-DGLEW_EGL)
  if (NOT OpenGL_EGL_FOUND)
    message (FATAL_ERROR "EGL library set but not found.")
  endif ()
  list (APPEND GLEW_LIBRARIES ${OPENGL_egl_LIBRARY})
endif ()

#### GLEW ####

set (GLEW_PUBLIC_HEADERS_FILES
  ${GLEW_DIR}/include/GL/wglew.h
  ${GLEW_DIR}/include/GL/glew.h
  ${GLEW_DIR}/include/GL/glxew.h
  ${GLEW_DIR}/include/GL/eglew.h
)
set (GLEW_SRC_FILES ${GLEW_DIR}/src/glew.c)

if (WIN32)
  list (APPEND GLEW_SRC_FILES ${GLEW_DIR}/glew.rc)
endif ()

add_library (glew SHARED ${GLEW_PUBLIC_HEADERS_FILES} ${GLEW_SRC_FILES})
set_target_properties (glew PROPERTIES COMPILE_DEFINITIONS "GLEW_BUILD" OUTPUT_NAME "${GLEW_LIB_NAME}" PREFIX "${DLL_PREFIX}"
                                       VERSION ${GLEW_VERSION}
                                       SOVERSION ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR})
target_include_directories (glew PUBLIC ${GLEW_DIR}/include)
add_library (glew_s STATIC ${GLEW_PUBLIC_HEADERS_FILES} ${GLEW_SRC_FILES})
set_target_properties (glew_s PROPERTIES COMPILE_DEFINITIONS "GLEW_STATIC" OUTPUT_NAME "${GLEW_LIB_NAME}" PREFIX lib)
target_include_directories (glew_s PUBLIC ${GLEW_DIR}/include)

if (MSVC)
  # add options from visual studio project
  target_compile_definitions (glew PRIVATE "GLEW_BUILD;VC_EXTRALEAN")
  target_compile_definitions (glew_s PRIVATE "GLEW_STATIC;VC_EXTRALEAN")
  if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(i?86|x86|x86_32)$")
	  target_link_libraries (glew LINK_PRIVATE -BASE:0x62AA0000)
  endif()
  # kill security checks which are dependent on stdlib
  target_compile_options (glew PRIVATE -GS-)
  target_compile_options (glew_s PRIVATE -GS-)
  # remove stdlib dependency
  target_link_libraries (glew LINK_PRIVATE -nodefaultlib -noentry)
  target_link_libraries (glew LINK_PRIVATE libvcruntime.lib)
  target_link_libraries (glew LINK_PRIVATE msvcrt.lib )
  string(REGEX REPLACE "/RTC(su|[1su])" "" CMAKE_C_FLAGS_DEBUG ${CMAKE_C_FLAGS_DEBUG})
elseif (WIN32 AND ((CMAKE_C_COMPILER_ID MATCHES "GNU") OR (CMAKE_C_COMPILER_ID MATCHES "Clang")))
  # remove stdlib dependency on windows with GCC and Clang (for similar reasons
  # as to MSVC - to allow it to be used with any Windows compiler)
  target_compile_options (glew PRIVATE -fno-builtin -fno-stack-protector)
  target_compile_options (glew_s PRIVATE -fno-builtin -fno-stack-protector)
  target_link_libraries (glew LINK_PRIVATE -nostdlib)
endif ()

if (BUILD_FRAMEWORK)
  set_target_properties(glew PROPERTIES
    FRAMEWORK TRUE
    FRAMEWORK_VERSION ${GLEW_VERSION}
    MACOSX_FRAMEWORK_IDENTIFIER net.sourceforge.glew
    MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${GLEW_VERSION}
    MACOSX_FRAMEWORK_BUNDLE_VERSION ${GLEW_VERSION}
    XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
    PUBLIC_HEADER "${GLEW_PUBLIC_HEADERS_FILES}"
    OUTPUT_NAME GLEW
  )
endif()

target_link_libraries (glew LINK_PUBLIC ${GLEW_LIBRARIES})
target_link_libraries (glew_s ${GLEW_LIBRARIES})

target_compile_definitions(glew_s INTERFACE "GLEW_STATIC")

if (BUILD_UTILS)
  set (GLEWINFO_SRC_FILES ${GLEW_DIR}/src/glewinfo.c)
  if (WIN32)
    list (APPEND GLEWINFO_SRC_FILES ${GLEW_DIR}/glewinfo.rc)
  endif ()
  add_executable (glewinfo ${GLEWINFO_SRC_FILES})
  if(BUILD_SHARED_LIBS)
     target_link_libraries (glewinfo glew)
  else()
     target_link_libraries (glewinfo glew_s)
  endif()
  if (NOT WIN32)
    target_link_libraries(glewinfo ${X11_LIBRARIES})
  endif ()

  set (VISUALINFO_SRC_FILES ${GLEW_DIR}/src/visualinfo.c)
  if (WIN32)
    list (APPEND VISUALINFO_SRC_FILES ${GLEW_DIR}/visualinfo.rc)
  endif ()
  add_executable (visualinfo ${VISUALINFO_SRC_FILES})
  if(BUILD_SHARED_LIBS)
     target_link_libraries (visualinfo glew)
  else()
     target_link_libraries (visualinfo glew_s)
  endif()
  if (NOT WIN32)
    target_link_libraries(visualinfo ${X11_LIBRARIES})
  endif ()
endif ()