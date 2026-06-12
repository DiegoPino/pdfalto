cmake_minimum_required(VERSION 3.10+)
project(pdfalto VERSION 0.6.0)

set(PDFALTO_VERSION_STR "${PROJECT_VERSION}")

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXE_LINKER_FLAGS "-no-pie")
set(CMAKE_BUILD_TYPE "Release")
set(CMAKE_FIND_LIBRARY_SUFFIXES ".a")

# Set the SDK path
set(CMAKE_OSX_SYSROOT /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk)

#--- look for fontconfig
if (NOT NO_FONTCONFIG)
    find_library(FONTCONFIG_LIBRARY
            NAMES fontconfig libfontconfig
            PATH_SUFFIXES lib64 lib
    )
    if (FONTCONFIG_LIBRARY)
        set(HAVE_FONTCONFIG TRUE)
        message(STATUS "Found fontconfig at ${FONTCONFIG_LIBRARY}")
    else ()
        set(HAVE_FONTCONFIG FALSE)
        set(FONTCONFIG_LIBRARY "")
    endif ()
else ()
    set(HAVE_FONTCONFIG FALSE)
    set(FONTCONFIG_LIBRARY "")
endif ()

#build xpdf
# See https://github.com/DiegoPino/xpdf-4.05/blob/main/CMakeLists.txt
set (XPDF_SUBDIR ${CMAKE_CURRENT_SOURCE_DIR}/xpdf-4.05)

set(IMAGE_SUBDIR ${CMAKE_CURRENT_SOURCE_DIR}/libs/image)

#set(PNG_SUBDIR ${CMAKE_CURRENT_SOURCE_DIR}/libs/image/png)

#set(ZLIB_SUBDIR ${CMAKE_CURRENT_SOURCE_DIR}/libs/image/zlib)

#set(ICU_SUBDIR ${CMAKE_CURRENT_SOURCE_DIR}/libs/icu)

#set(LIBXML_SUBDIR ${CMAKE_CURRENT_SOURCE_DIR}/libs/libxml)

#set(FREETYPE_SUBDIR ${CMAKE_CURRENT_SOURCE_DIR}/libs/freetype)


# Use Alpine Statically Installed libraries + headers
# apk add autoconf automake libtool build-base dd alpine-sdk cmake
# apk add icu-static libxml2-static zlib-static libpng-static freetype-static  libxml2-dev icu-dev  zlib-dev libpng-dev  freetype-dev  fontconfig-dev xz-dev xz-static brotli-dev brotli-static bzip2-dev bzip2-static


find_package(PkgConfig REQUIRED)

find_package(LibXml2 REQUIRED)
include_directories(${LIBXML2_INCLUDE_DIR})

find_package(ZLIB REQUIRED)
include_directories(${ZLIB_INCLUDE_DIR})

find_package(PNG REQUIRED)
include_directories(${PNG_INCLUDE_DIR})

find_package(Freetype REQUIRED)
include_directories(${Freetype_INCLUDE_DIR})

find_package(ICU REQUIRED COMPONENTS uc data i18n io)
include_directories(${ICU_INCLUDE_DIR})

find_package(LibLZMA REQUIRED)
include_directories(${LibLZMA_INCLUDE_DIR})

find_package(BZip2 REQUIRED)
include_directories(${BZip2_INCLUDE_DIR})

find_package(Brotli REQUIRED)
include_directories(${Brotli_INCLUDE_DIR})

set(FREETYPE_INCLUDE_DIR_ft2build ${FREETYPE_INCLUDE_DIR})
set(FREETYPE_INCLUDE_DIR_freetype_freetype ${FREETYPE_INCLUDE_DIR})
set(FREETYPE_INCLUDE_DIR ${FREETYPE_INCLUDE_DIR})
#set(PNG_FOUND TRUE)


if (APPLE)
    message("Using macos settings.")
    #set(CMAKE_CXX_STANDARD_LIBRARIES "-static-compiler-rt -static-libc++")
    set(OSSUFFIX "mac")

    include(CheckCCompilerFlag)
    check_c_compiler_flag("-arch arm64" arm64Supported)
    message("arm64Supported=${arm64Supported}")


    IF (${arm64Supported} MATCHES 1)
        set(ARCHSUFFIX "arm64")
        set(CMAKE_REQUIRED_LINK_OPTIONS "-arch;arm64")
        MESSAGE(STATUS "Check if the building for MacOS ARM64")
    ELSE ()
        set(ARCHSUFFIX "64")
        set(C_INCLUDE_PATH "/usr/local/include")
        set(CPLUS_INCLUDE_PATH "/usr/local/include")
        MESSAGE(STATUS "Check if the building for MacOS X86_64")
    ENDIF ()

elseif (CYGWIN OR MSVC OR WIN32 OR MINGW)
    message("Using windows settings.")
    set(CMAKE_CXX_STANDARD_LIBRARIES "-static-libgcc -static-libstdc++")
    set(OSSUFFIX "windows")
elseif (UNIX)
   set(CMAKE_CXX_STANDARD_LIBRARIES "-static-libgcc -static-libstdc++")
  
    set(OSSUFFIX "linux")
endif ()

if (${OSSUFFIX} MATCHES "windows" OR ${OSSUFFIX} MATCHES "linux")
    if (CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64" OR CMAKE_SYSTEM_PROCESSOR MATCHES "arm64")
        set(ARM64 TRUE)
    else ()
        set(ARM64 FALSE)
    endif ()

    if ("${CMAKE_SIZEOF_VOID_P}" EQUAL "8")
        if(ARM64)
            MESSAGE("++ 64 bit ARM architecture")
            set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=armv8-a")
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=armv8-a")
            set(ARCHSUFFIX "arm64")
        else()
            MESSAGE("++ 64 bit architecture")
            set(ARCHSUFFIX "64")
        endif()
    else ()
        MESSAGE("++ 32 bit architecture")
        set(ARCHSUFFIX "32")
    endif ()
endif ()

MESSAGE("Final os/arch: " ${OSSUFFIX}/${ARCHSUFFIX})



set(XPDF_BUILD_DIR ${XPDF_SUBDIR}/build)

add_subdirectory(${XPDF_SUBDIR})

set(SOURCE_FILES
        src/AnnotsXrce.cc
        src/AnnotsXrce.h
        src/ConstantsUtils.cc
        src/ConstantsUtils.h
        src/ConstantsXML.cc
        src/ConstantsXML.h
        src/Parameters.cc
        src/Parameters.h
        src/PDFDocXrce.cc
        src/PDFDocXrce.h
        src/whereami.c
        src/whereami.h
        src/pdfalto.cc
        src/XmlAltoOutputDev.cc
        src/XmlAltoOutputDev.h
)

add_executable(pdfalto ${SOURCE_FILES})

target_compile_definitions(pdfalto PRIVATE PDFALTO_VERSION_STR=\"${PDFALTO_VERSION_STR}\")
 #   message("Using Alpine linux settings.")
 #   add_library(alpine_std_libs INTERFACE)
 #   target_link_libraries(alpine_std_libs INTERFACE -lc++) 
 #   target_link_libraries(pdfalto PRIVATE alpine_std_libs)

if (HAVE_PAPER_H)
    if (HAVE_FONTCONFIG)
        target_link_libraries(pdfalto  PNG::PNG LibXml2::LibXml2 ICU::uc ICU::data ICU::i18n ZLIB::ZLIB splash Freetype::Freetype xpdf goo fofi dl LibLZMA::LibLZMA Brotli::Brotli BZip2::BZip2 ${PAPER_LIBRARY} ${FONTCONFIG_LIBRARY} pthread)
    else ()
        target_link_libraries(pdfalto  PNG::PNG LibXml2::LibXml2 ICU::uc ICU::data ICU::i18n ZLIB::ZLIB splash Freetype::Freetype xpdf goo fofi dl LibLZMA::LibLZMA Brotli::Brotli $BZip2::BZip2 {PAPER_LIBRARY} pthread)
    endif ()
else ()
    if (HAVE_FONTCONFIG)
        target_link_libraries(pdfalto  PNG::PNG LibXml2::LibXml2 ICU::uc ICU::data ICU::i18n ZLIB::ZLIB splash Freetype::Freetype xpdf goo fofi dl LibLZMA::LibLZMA Brotli::Brotli BZip2::BZip2 ${FONTCONFIG_LIBRARY} pthread)
    else ()
        target_link_libraries(pdfalto PNG::PNG LibXml2::LibXml2 ICU::uc ICU::data ICU::i18n ZLIB::ZLIB splash Freetype::Freetype xpdf goo fofi dl LibLZMA::LibLZMA Brotli::Brotli BZip2::BZip2 pthread)
    endif ()
endif ()


target_include_directories(
        pdfalto
        PRIVATE ${XML_INCLUDE_DIRS}
        PRIVATE ${Freetype_INCLUDE_DIR}
        PRIVATE ${PNG_INCLUDE_DIRS}
        PRIVATE ${ZLIB_INCLUDE_DIRS}
        PRIVATE ${ICU_INCLUDE_DIRS}
        PRIVATE ${LibLZMA_INCLUDE_DIRS}
        PRIVATE ${Brotli_INCLUDE_DIRS}
        PRIVATE ${BZip2_INCLUDE_DIRS}
        PRIVATE ${XPDF_SUBDIR}
        PRIVATE ${XPDF_SUBDIR}/goo
        PRIVATE ${XPDF_SUBDIR}/fofi
        PRIVATE ${XPDF_SUBDIR}/xpdf
        PRIVATE ${XPDF_BUILD_DIR}
        
)
