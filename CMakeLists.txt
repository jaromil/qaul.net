cmake_minimum_required (VERSION 2.8.11)

project (qaul)

if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
	if (APPLE)
		set (CMAKE_INSTALL_PREFIX "/Library/qaul.net" CACHE PATH "qaul.net install prefix" FORCE)
	endif ()
endif (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake")

include (ExternalProject)

# The version number.
set (Qaul_VERSION_MAJOR 1)
set (Qaul_VERSION_MINOR 0)
set (Qaul_VERSION_PATCH 0)
set (Qaul_VERSION_SUFFIX "beta5")
set (Qaul_VERSION "${Qaul_VERSION_MAJOR}.${Qaul_VERSION_MINOR}.${Qaul_VERSION_PATCH}.${Qaul_VERSION_SUFFIX}")

# install configuration
set (Qaul_DESKTOPDIR "/usr/share/applications" CACHE STRING "choose install location for the .desktop file")
set (Qaul_ICONDIR "${CMAKE_INSTALL_PREFIX}/lib/qaul/icons" CACHE STRING "choose install location for the aplication icon file(s)")
set (Qaul_BINDIR "NONE" CACHE STRING "if set a wrapper shell script will be installed here")

# What to build
# Determine the operating system
set (ALL_PORTS ANDROID OPENWRT WINDOWS OSX IOS LINUX)
STRING (TOUPPER "${PORT}" PORT)
set (PORT "AUTO" CACHE STRING "choose which qaul port to build (one of ${ALL_PORTS})")

if (${PORT} STREQUAL "AUTO")
  if (UNIX)
    if (APPLE)
      set (PORT "OSX")
    else ()
      set (PORT "LINUX")
    endif ()
  elseif (CMAKE_SYSTEM_NAME MATCHES "Windows")
    set (PORT "WINDOWS")
  else ()
    message(FATAL_ERROR "Unknown OS '${CMAKE_SYSTEM_NAME}'")
  endif ()
endif ()

list(FIND ALL_PORTS ${PORT} RET)
if (${RET} EQUAL -1)
  message(FATAL_ERROR "Please choose which qaul port to build (one of ${ALL_PORTS})")
endif ()

# determine the GUI
set(ALL_GUIS CLI NATIVE GTK QT5)
STRING (TOUPPER "${GUI}" GUI)
set(GUI "DEFAULT" CACHE STRING "Choose which qaul GUI to build (one of ${ALL_GUIS})")

# readjust defaults
if (${GUI} STREQUAL "DEFAULT")
  if ("${PORT}" STREQUAL "LINUX")
    set (GUI GTK)
  else ()
    set (GUI NATIVE)
  endif ()
endif ()

# set the name for the ui executable
if (${GUI} STREQUAL "GTK")
    set (GUINAME "qaul-gtk")
elseif (${GUI} STREQUAL "QT5")
    set (GUINAME "qaul-qt5")
elseif (${GUI} STREQUAL "CLI")
    set (GUINAME "qaul-cli")
else ()
    set (GUINAME "qaul")
endif ()

if (${PORT} STREQUAL "WINDOWS")
  set (GUINAME "${GUINAME}.exe")
endif ()

list(FIND ALL_GUIS ${GUI} RET)
if (${RET} EQUAL -1)
  message(FATAL_ERROR "Please choose which qaul GUI to build (one of ${ALL_GUIS})")
endif ()

# What package to build
set(ALL_PKGFORMAT AUTO NSIS TGZ RPM DEB)
set(PKGFORMAT "AUTO" CACHE STRING "select what type of package should be build (one of ${ALL_PKGFORMAT})")

# Cpack general Installer settings
set(CPACK_RESOURCE_FILE_LICENSE ${PROJECT_SOURCE_DIR}/Licenses/qaul.net_license.txt)
set(CPACK_PACKAGE_VENDOR "qaul.net community")
set(CPACK_PACKAGE_VERSION_MAJOR ${Qaul_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${Qaul_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${Qaul_VERSION_PATCH})
set(CPACK_PACKAGE_VERSION_SUFFIX ${Qaul_VERSION_SUFFIX})
set(CPACK_PACKAGE_VERSION ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}-${CPACK_PACKAGE_VERSION_SUFFIX})

# configure a header file to pass some of the CMake settings
# to the source code
configure_file (
  "${PROJECT_SOURCE_DIR}/include/QaulConfig.h.in"
  "${PROJECT_BINARY_DIR}/include/QaulConfig.h"
)

# add the binary tree to the search path for include files
# so that we will find QaulConfig.h
include_directories ("${PROJECT_BINARY_DIR}/include")

if (${PORT} STREQUAL "LINUX")
    include(cmake/BuildLinux.cmake)
elseif (${PORT} STREQUAL "ANDROID")
    include(cmake/BuildAndroid.cmake)
elseif (${PORT} STREQUAL "OPENWRT")
    include(cmake/BuildOpenWrt.cmake)
elseif (${PORT} STREQUAL "OSX")
    include(cmake/BuildOSX.cmake)
elseif (${PORT} STREQUAL "WINDOWS")
    include(cmake/BuildWindows.cmake)
else ()
    message(FATAL_ERROR "Dont know how to build '${PORT}'")
endif ()

add_subdirectory (third_party)

if (NOT ${PORT} STREQUAL "ANDROID")
    add_subdirectory (src/libqaul)
endif ()

if (${GUI} STREQUAL "NATIVE")
    if (${PORT} STREQUAL "OSX")
	add_subdirectory (src/client/osx)
    elseif (${PORT} STREQUAL "WINDOWS")
	# add_subdirectory (src/client/win)
    elseif (${PORT} STREQUAL "ANDROID")
	# have its own
    else ()
	message(FATAL_ERROR "Dont know how to build UI '${GUI}' on port '${PORT}'")
    endif ()
elseif (${GUI} STREQUAL "CLI")
    add_subdirectory (src/client/cli)
elseif (${GUI} STREQUAL "GTK")
    add_subdirectory (src/client/gtk)
elseif (${GUI} STREQUAL "QT5")
    add_subdirectory (src/client/qt5)
else ()
    message(FATAL_ERROR "Dont know how to build ui '${GUI}'")
endif ()

# final status summary report
message(STATUS "#############################################")
message(STATUS "  qaul seems to be successfully configured.")
message(STATUS "    Building for platform: ${PORT}")
message(STATUS "    Building GUI: ${GUI}")
message(STATUS "    Arm/Raspberry patches enabled: ${Raspberry}")
message(STATUS "    qaul executable: ${GUINAME}")
message(STATUS "#############################################")
