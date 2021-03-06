cmake_minimum_required(VERSION 2.6)

set(SDK8_ROOT ../../../)

add_definitions(-DDIB_INTERNAL_MONITORING=1)

add_subdirectory(../../LinuxDriverLibrary/ ldl/)

# Add proper rpath configuration
option(ENABLE_PACKAGING "Enable debian packaging" FALSE)

if (ENABLE_PACKAGING)
    set(CMAKE_SKIP_BUILD_RPATH  TRUE)
    set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
    set(CMAKE_INSTALL_RPATH "/usr/local/lib")
    set(CMAKE_INSTALL_RPATH_USE_LINK_PATH FALSE)
endif(ENABLE_PACKAGING)

include_directories(../../../User/Common .)

#file(GLOB SRC_PLATFORM *.c)
file(GLOB SRC ../*.c)

set(DEBUG_LIB OFF CACHE BOOL "Enable debug-library.")
if(DEBUG_LIB)
    add_definitions(-DDEBUG_LIB)

    add_library(Debug STATIC IMPORTED)
    set_property(TARGET Debug PROPERTY IMPORTED_LOCATION "../Debug/Debug.a")

    set(EXTRA_LIB ${EXTRA_LIB} Debug)
endif()

set(DEBUG_GDB ON CACHE BOOL "Enable GDB debug support.")
if(DEBUG_GDB)
    add_definitions(-DDEBUG)

    message("GDB enabled:
    1) make menu -> GDB enabled,
    2) DibConfig.h -> DIB_INTERNAL_DEBUG = 1,
    3) DibDriverDragonflyIntDbg.h -> DF_GDB_STUB = 1,
    4) ddd -debugger sparc-elf-gdb")

endif()

add_executable(dta ${SRC_PLATFORM} ${SRC})
target_link_libraries(dta m pthread DibDriverLibrary ${EXTRA_LIB})

#add_executable(test_tune test_tune.c)
#target_link_libraries(test_tune m pthread DibDriverLibrary)

install(TARGETS dta RUNTIME DESTINATION bin)

# Get SDK version
file(STRINGS "${SDK8_ROOT}/Driver/Common/DibDriverConstants.h" version REGEX "DIB_SET_VER\\([0-9]+")
string(REGEX MATCH "[0-9 ]+,[0-9 ]+,[0-9 ]+" VERSION_STRING ${version})
string(REGEX MATCH "[0-9]+" VERSION_MAJOR ${version})
string(REGEX REPLACE " " "" VERSION_STRING ${VERSION_STRING})
string(REGEX REPLACE , . VERSION_STRING ${VERSION_STRING})

if(ENABLE_PACKAGING)
    set(CPACK_GENERATOR "DEB")
    set(CPACK_PACKAGE_NAME "dibcom-sdk-${VERSION_STRING}")
    set(CPACK_PACKAGE_VERSION ${VERSION_STRING})
    execute_process(COMMAND dpkg --print-architecture OUTPUT_VARIABLE CPACK_DEBIAN_PACKAGE_ARCHITECTURE ERROR_QUIET OUTPUT_STRIP_TRAILING_WHITESPACE)
    set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME}_${CPACK_PACKAGE_VERSION}_${CPACK_DEBIAN_PACKAGE_ARCHITECTURE}")
    set(CPACK_PACKAGING_INSTALL_PREFIX "/usr/local")
    set(CPACK_DEBIAN_PACKAGE_CONTROL_EXTRA "${CMAKE_CURRENT_SOURCE_DIR}/debian/control;${CMAKE_CURRENT_SOURCE_DIR}/debian/postinst;")
    set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Cyril Laury <cyril.laury@dibcom.com>")
    # Override missing functionality for CPack Debian Generation
    file(WRITE  ${CMAKE_CURRENT_SOURCE_DIR}/debian/control "Package: ${CPACK_PACKAGE_NAME}\n")
    file(APPEND ${CMAKE_CURRENT_SOURCE_DIR}/debian/control "Version: ${CPACK_PACKAGE_VERSION}\n")
    file(APPEND ${CMAKE_CURRENT_SOURCE_DIR}/debian/control "Architecture: ${CPACK_DEBIAN_PACKAGE_ARCHITECTURE}\n")
    file(APPEND ${CMAKE_CURRENT_SOURCE_DIR}/debian/control "Maintainer: Cyril Laury <cyril.laury@dibcom.com\n")
    file(APPEND ${CMAKE_CURRENT_SOURCE_DIR}/debian/control "Priority: Optional\n")
    file(APPEND ${CMAKE_CURRENT_SOURCE_DIR}/debian/control "Section: devel\n")
    file(APPEND ${CMAKE_CURRENT_SOURCE_DIR}/debian/control "Description: DiBcom Software Development Kit\n")
    file(APPEND ${CMAKE_CURRENT_SOURCE_DIR}/debian/control "Depends: dibcom-usb-kernel-module\n")
    file(APPEND ${CMAKE_CURRENT_SOURCE_DIR}/debian/control "Provides: dibcom-sdk-driver\n")
    file(APPEND ${CMAKE_CURRENT_SOURCE_DIR}/debian/control "Conflicts: dibcom-sdk-driver\n")
    file(APPEND ${CMAKE_CURRENT_SOURCE_DIR}/debian/control "Replaces: dibcom-sdk-driver\n")
    include(CPack)
else(ENABLE_PACKAGING)
    execute_process(COMMAND rm -f ${CMAKE_CURRENT_SOURCE_DIR}/cmake_install.cmake)
    execute_process(COMMAND rm -f ${CMAKE_CURRENT_SOURCE_DIR}/CPackConfig.cmake)
    execute_process(COMMAND rm -f ${CMAKE_CURRENT_SOURCE_DIR}/CPackSourceConfig.cmake)
    execute_process(COMMAND rm -rf ${CMAKE_CURRENT_SOURCE_DIR}/_CPack_Packages)
endif(ENABLE_PACKAGING)
