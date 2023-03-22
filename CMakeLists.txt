cmake_minimum_required(VERSION 3.13)
cmake_policy(SET CMP0048 NEW)

project(gSDK
    VERSION 0.0.0
    LANGUAGES CXX
)

include(GNUInstallDirs)

set(GSDK_LIB_SRC
    src/gimbal_interface.cpp
    src/gimbal_protocol.cpp
    src/gimbal_protocol_v1.cpp
    src/gimbal_protocol_v2.cpp
    src/serial_port.cpp
)

find_package( Threads )

add_library(gSDK SHARED
    ${GSDK_LIB_SRC}
)

set_target_properties(gSDK PROPERTIES
    SOVERSION ${PROJECT_VERSION}
)

###############################################################################
#### INCLUDES #################################################################
###############################################################################

target_include_directories(gSDK PUBLIC
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>"
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src/mavlink/include/mavlink/v2.0>"
    "$<INSTALL_INTERFACE:include/gSDK>"
    "$<INSTALL_INTERFACE:include/gSDK/mavlink/v2.0>"
)

###############################################################################
#### DEPENDENCIES #############################################################
###############################################################################

target_link_libraries(gSDK
    PUBLIC Threads::Threads
)

###############################################################################
#### GENERATE OUTPUT ##########################################################
###############################################################################

set(mavlinkHeadersSrc
    src/mavlink/include/mavlink/v2.0/checksum.h
    src/mavlink/include/mavlink/v2.0/mavlink_conversions.h
    src/mavlink/include/mavlink/v2.0/mavlink_get_info.h
    src/mavlink/include/mavlink/v2.0/mavlink_helpers.h
    src/mavlink/include/mavlink/v2.0/mavlink_sha256.h
    src/mavlink/include/mavlink/v2.0/mavlink_types.h
    src/mavlink/include/mavlink/v2.0/protocol.h
)

file(GLOB_RECURSE mavlinkHeadersOtherSrc
    LIST_DIRECTORIES TRUE
    RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}"
    src/mavlink/include/mavlink/v2.0/ardupilotmega/*.h
    src/mavlink/include/mavlink/v2.0/common/*.h
    src/mavlink/include/mavlink/v2.0/minimal/*.h
    src/mavlink/include/mavlink/v2.0/uAvionix/*.h
    src/mavlink/include/mavlink/v2.0/icarous/*.h
)

if (CMAKE_VERSION VERSION_GREATER_EQUAL 3.23)
    list(APPEND mavlinkHeadersSrc "${mavlinkHeadersOtherSrc}")
    target_sources(gSDK PUBLIC
        FILE_SET mavlinkHeaders
        TYPE HEADERS
        BASE_DIRS src/mavlink/include/mavlink/v2.0
        FILES ${mavlinkHeadersSrc}
    )
endif()

if (CMAKE_VERSION VERSION_GREATER_EQUAL 3.23)
    install(TARGETS gSDK
            EXPORT gSDKTargets
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
            FILE_SET mavlinkHeaders DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/gSDK/mavlink/v2.0
            INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/gSDK
    )
else()
    install(TARGETS gSDK
            EXPORT gSDKTargets
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
            INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/gSDK
    )
    install(FILES ${mavlinkHeadersSrc}
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/gSDK/mavlink/v2.0 
    )
    install(DIRECTORY src/mavlink/include/mavlink/v2.0/ardupilotmega 
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/gSDK/mavlink/v2.0/
        FILES_MATCHING PATTERN "*.h"
    )
    install(DIRECTORY src/mavlink/include/mavlink/v2.0/common
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/gSDK/mavlink/v2.0/
        FILES_MATCHING PATTERN "*.h"
    )
    install(DIRECTORY src/mavlink/include/mavlink/v2.0/minimal
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/gSDK/mavlink/v2.0/
        FILES_MATCHING PATTERN "*.h"
    )
    install(DIRECTORY src/mavlink/include/mavlink/v2.0/uAvionix
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/gSDK/mavlink/v2.0/
        FILES_MATCHING PATTERN "*.h"
    )
    install(DIRECTORY src/mavlink/include/mavlink/v2.0/icarous
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/gSDK/mavlink/v2.0/
        FILES_MATCHING PATTERN "*.h"
    )
endif()

install(FILES src/gimbal_interface.h src/gimbal_protocol_v1.h src/gimbal_protocol_v2.h src/gimbal_protocol.h src/serial_port.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/gSDK
)

add_executable(gSDKTestApp
    mavlink_control.cpp
)

target_link_libraries(gSDKTestApp PRIVATE gSDK)

install(TARGETS gSDKTestApp
    EXPORT gSDKTargets
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

export(EXPORT gSDKTargets
       FILE "${CMAKE_CURRENT_BINARY_DIR}/gSDKTargets.cmake"
)

install(EXPORT gSDKTargets
        FILE gSDKTargets.cmake
        DESTINATION  ${CMAKE_INSTALL_LIBDIR}/cmake/gSDK
)

include(CMakePackageConfigHelpers)

configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in
    "${CMAKE_CURRENT_BINARY_DIR}/gSDKConfig.cmake"
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/gSDK
)

install(FILES
          "${CMAKE_CURRENT_BINARY_DIR}/gSDKConfig.cmake"
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/gSDK
)

add_subdirectory("${CMAKE_CURRENT_SOURCE_DIR}/examples")

###############################################################################
#### GENERATE PACKAGES ########################################################
###############################################################################

if (CMAKE_SYSTEM_NAME MATCHES Linux)
    if (CMAKE_VERSION VERSION_LESS 3.25)
        # The LINUX variable is defined since CMake 3.25
        set(LINUX TRUE)
    endif()
endif()

if (LINUX)
   if (EXISTS "/etc/debian_version")
      SET (CPACK_GENERATOR "DEB")
      SET (CPACK_DEBIAN_PACKAGE_MAINTAINER "Gremsy")
      SET (CPACK_DEBIAN_FILE_NAME DEB-DEFAULT)
      SET (CPACK_DEBIAN_PACKAGE_SHLIBDEPS ON)
   elseif(EXISTS "/etc/redhat-release")
      SET (CPACK_GENERATOR "RPM")
      SET (CPACK_RPM_FILE_NAME RPM-DEFAULT)
      SET (CPACK_RPM_PACKAGE_AUTOREQ 0)
      SET (CPACK_RPM_EXCLUDE_FROM_AUTO_FILELIST_ADDITION /usr/lib64/cmake)
   else()
      SET (CPACK_GENERATOR "ZIP")
      SET (CPACK_PACKAGE_FILE_NAME "gSDK_${PROJECT_VERSION}_${CMAKE_SYSTEM_NAME}")
   endif()
else()
   SET (CPACK_GENERATOR "ZIP")
   SET (CPACK_PACKAGE_FILE_NAME "gSDK_${PROJECT_VERSION}_${CMAKE_SYSTEM_NAME}")
endif()
SET (CPACK_PACKAGE_VENDOR "Gremsy")

include(CPack)
