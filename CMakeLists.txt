# Copyright (C) 2016-2017  Fulvio Benini
# This file is part of Scid (Shane's Chess Information Database).
#
# Scid is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation.
#
# Scid is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with Scid. If not, see <http://www.gnu.org/licenses/>.

cmake_minimum_required(VERSION 3.20)
project(scid)
set(CPACK_PACKAGE_VERSION 5.2.202603)
set(
  CPACK_PACKAGE_DESCRIPTION_SUMMARY
  "chess database application with play and training functionality"
)
set(CPACK_DEBIAN_PACKAGE_DEPENDS "tk8.6 (>= 8.6.0)")
set(CPACK_DEBIAN_PACKAGE_SHLIBDEPS ON)
include(CPack)

# Visual Studio generator settings
if(MSVC)
  set(CMAKE_GENERATOR_PLATFORM "x64" CACHE STRING "")
  set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
endif()

set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD 20 CACHE STRING "")
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CXX_FLAGS)
  set(CMAKE_BUILD_TYPE "Release")
endif()

if(MSVC)
  add_definitions(/D_CRT_SECURE_NO_WARNINGS /D_SCL_SECURE_NO_WARNINGS /D_CRT_NONSTDC_NO_DEPRECATE)
  # Standards compliance flags applied to all targets
  add_compile_options(/permissive- /Zc:__cplusplus /utf-8 /MP)
  # To run/debug using Visual Studio set "scid" as startup project and add:
  # Command Arguments: ../tcl/start.tcl
  # Environment:       PATH=C:\tcl\bin
endif()

find_package(Threads)

# Find TCL with helpful error messages
set(TCL_POSSIBLE_PATHS
  "C:/Tcl"
  "C:/Program Files/Tcl"
  "C:/ActiveTcl"
  "$ENV{ProgramFiles}/Tcl"
)
# Provide hints to find_package before calling it
foreach(TCL_PATH ${TCL_POSSIBLE_PATHS})
  if(EXISTS "${TCL_PATH}/include/tcl.h")
    list(APPEND CMAKE_PREFIX_PATH "${TCL_PATH}")
    break()
  endif()
endforeach()
find_package(TCL)

if(NOT TCL_INCLUDE_PATH)
  foreach(TCL_PATH ${TCL_POSSIBLE_PATHS})
    if(EXISTS "${TCL_PATH}/include/tcl.h")
      set(TCL_INCLUDE_PATH "${TCL_PATH}/include")
      message(STATUS "Found TCL include at: ${TCL_INCLUDE_PATH}")
      break()
    endif()
  endforeach()
endif()

if(NOT TCL_LIBRARY)
  foreach(TCL_PATH ${TCL_POSSIBLE_PATHS})
    # Prefer tcl90.lib (v9), fall back to tcl86.lib (v8.6)
    foreach(TCL_LIB_NAME "tcl90.lib" "tcl86t.lib" "tcl86.lib")
      if(EXISTS "${TCL_PATH}/lib/${TCL_LIB_NAME}")
        set(TCL_LIBRARY "${TCL_PATH}/lib/${TCL_LIB_NAME}")
        message(STATUS "Found TCL library at: ${TCL_LIBRARY}")
        break()
      endif()
    endforeach()
    if(TCL_LIBRARY)
      break()
    endif()
  endforeach()
endif()

if(NOT TCL_INCLUDE_PATH OR NOT TCL_LIBRARY)
  message(FATAL_ERROR
    "\n"
    "================================================================\n"
    "TCL/Tk is required to build Scid but was not found.\n"
    "\n"
    "Please install Tcl/Tk:\n"
    "  - MagicSplat: choco install magicsplat-tcl-tk (installs to C:/Program Files/Tcl)\n"
    "  - Or from:    https://www.tcl.tk/software/tcltk/\n"
    "\n"
    "After installation, specify the path manually if needed:\n"
    "  cmake -DTCL_INCLUDE_PATH=\"C:/Program Files/Tcl/include\" \\\n"
    "        -DTCL_LIBRARY=\"C:/Program Files/Tcl/lib/tcl90.lib\" \\\n"
    "        -B build\n"
    "================================================================\n"
  )
endif()

message(STATUS "Using TCL include: ${TCL_INCLUDE_PATH}")
message(STATUS "Using TCL library: ${TCL_LIBRARY}")

# polyglot
file(GLOB POLYGLOT_SRC src/polyglot/*.cpp)
add_library(polyglot ${POLYGLOT_SRC})
if(MSVC)
  target_compile_options(polyglot PRIVATE /W4 /WX- /wd4100 /wd4127 /wd4244 /wd4267 /wd4458 /wd4459)
endif()

# scid
file(GLOB SCID_SRC src/*.h src/*.cpp)
if(MSVC)
  add_executable(scid WIN32 ${SCID_SRC} resources/win/scid.rc resources/win/scid.manifest)
  target_link_options(scid PRIVATE /ENTRY:mainCRTStartup)
else()
  add_executable(scid ${SCID_SRC})
endif()
set_property(TARGET scid PROPERTY INTERPROCEDURAL_OPTIMIZATION_RELEASE True)
if(MSVC)
  target_compile_options(scid PRIVATE /W4 /WX- /wd4100 /wd4127 /wd4244 /wd4267 /wd4458 /wd4459)
endif()
target_include_directories(scid PRIVATE ${TCL_INCLUDE_PATH})
target_link_libraries(scid PRIVATE polyglot ${CMAKE_THREAD_LIBS_INIT} ${TCL_LIBRARY})

install(TARGETS scid DESTINATION scid)
file(GLOB ECO_FILES *.eco)
install(FILES ${ECO_FILES} DESTINATION scid)
install(DIRECTORY bitmaps DESTINATION scid)
install(DIRECTORY bitmaps2 DESTINATION scid)
install(DIRECTORY books DESTINATION scid)
install(DIRECTORY html DESTINATION scid)
install(DIRECTORY img DESTINATION scid)
install(DIRECTORY scripts DESTINATION scid)
install(DIRECTORY sounds DESTINATION scid)
install(DIRECTORY tcl DESTINATION scid)


# engine phalanx
file(GLOB PHALANX_SRC engines/phalanx-scid/*.c)
add_executable(phalanx-scid ${PHALANX_SRC})
if(MSVC)
  target_compile_options(phalanx-scid PRIVATE /W0)
else()
  target_compile_options(phalanx-scid PRIVATE -w)
endif()
install(TARGETS phalanx-scid DESTINATION bin)


option(GTEST "Build unit tests" OFF)
if(GTEST)
  enable_testing()
  include(CTest)
  add_subdirectory(gtest)
endif()
