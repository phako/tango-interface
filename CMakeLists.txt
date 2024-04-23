cmake_minimum_required(VERSION 3.16)

project(tango-interface)

set(${PROJECT_NAME}_MAJOR_VERSION 01)
set(${PROJECT_NAME}_MINOR_VERSION 00)
set(${PROJECT_NAME}_PATCH_VERSION 00)

include(${CMAKE_SOURCE_DIR}/cmake/set_version_numbers.cmake)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake/Modules)
include(${CMAKE_SOURCE_DIR}/cmake/set_default_flags.cmake)
include(${CMAKE_SOURCE_DIR}/cmake/set_default_build_to_release.cmake)

find_program(LSB_RELEASE lsb_release)

if (LSB_RELEASE)
    execute_process(COMMAND ${LSB_RELEASE} "-c" OUTPUT_VARIABLE CODENAME_RAW)
    string(REPLACE ":" ";" CODENAME_RAW ${CODENAME_RAW})
    list(GET CODENAME_RAW 1 CODENAME)
    string(STRIP ${CODENAME} CODENAME)
    if(CODENAME STREQUAL "n/a")
        # Hello, are you Tumbleweed
        execute_process(COMMAND ${LSB_RELEASE} "-d" OUTPUT_VARIABLE LSB_DESCRIPTION)
        if (LSB_DESCRIPTION MATCHES "umbleweed")
            set(CODENAME "tumbleweed")
        endif()
    endif()
else()
    set(CODENAME "Unknown")
endif()

message("Distribution codename is ${CODENAME}")
set(TANGO_GENERATOR Ninja)
if (CODENAME STREQUAL "focal")
    set(IDL_VERSION "5.1.0")
    set(ORB_VERSION "4.3.0")
    set(ORB_HASH "976045a2341f4e9a85068b21f4bd928993292933eeecefea372db09e0219eadd")
    set(TANGO_VERSION "9.3.6")
    set(TANGO_GENERATOR "Unix Makefiles")
elseif (CODENAME STREQUAL "tumbleweed")
    set(IDL_VERSION "6.0-alpha2")
    set(ORB_VERSION "4.3.2")
    set(ORB_HASH "1c745330d01904afd7a1ed0a5896b9a6e53ac1a4b864a48503b93c7eecbf1fa8")
    set(TANGO_VERSION "a0514090977053c336f0deedaf4eecc6a651f197")
else ()
    set(IDL_VERSION "5.1.2")
    set(ORB_VERSION "4.3.2")
    set(ORB_HASH "1c745330d01904afd7a1ed0a5896b9a6e53ac1a4b864a48503b93c7eecbf1fa8")
    set(TANGO_VERSION "9.5.0")
endif()

message("Selected versions for ${CODENAME}")
message("IDL: ${IDL_VERSION}, ORB: ${ORB_VERSION} (with sha256 ${ORB_HASH}), tango: ${TANGO_VERSION}")

set(INTERMEDIATE_INSTALL_FOLDER ${CMAKE_BINARY_DIR}/temp_install)

include(ExternalProject)
ExternalProject_add(tango_idl_external
    GIT_REPOSITORY "https://gitlab.com/tango-controls/tango-idl.git"
    GIT_TAG "${IDL_VERSION}"
    CMAKE_GENERATOR Ninja
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
)

ExternalProject_add(omniORB_external
    URL "https://sourceforge.net/projects/omniorb/files/omniORB/omniORB-${ORB_VERSION}/omniORB-${ORB_VERSION}.tar.bz2"
    URL_HASH SHA256=${ORB_HASH}
    CONFIGURE_COMMAND <SOURCE_DIR>/configure --prefix=${CMAKE_INSTALL_PREFIX}
)

set(ENV{PYTHONPATH} "${INTERMEDIATE_INSTALL_FOLDER}${CMAKE_INSTALL_PREFIX}/python2.7/dist-packages:$ENV{PYTHONPOATH}:${tango_idl_path}/lib/pkgconfig")

# We drop cpp_test_suite for old tango since it depends on python being python2
ExternalProject_add(cpptango_external
    GIT_REPOSITORY "https://gitlab.com/tango-controls/cppTango.git"
    GIT_TAG "${TANGO_VERSION}"
    DEPENDS tango_idl_external omniORB_external
    PATCH_COMMAND "sed" "--in-place" "-e" "/add_subdirectory(\"cpp_test_suite\")'/d" <SOURCE_DIR>/CMakeLists.txt
    CMAKE_COMMAND PYTHONPATH=${INTERMEDIATE_INSTALL_FOLDER}${CMAKE_INSTALL_PREFIX}/lib/python2.7/dist-packages ${CMAKE_COMMAND}
    CMAKE_GENERATOR ${TANGO_GENERATOR}
    CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
        -DBUILD_TESTING=Off
)

# Fake install target
install(FILES README.md DESTINATION ${CMAKE_INSTALL_PREFIX}/share/doc/tango-interface)
