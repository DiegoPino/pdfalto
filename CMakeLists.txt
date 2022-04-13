  cmake_minimum_required(VERSION 3.5+)
  project(pdfalto)

  set(CMAKE_CXX_STANDARD 11)
  set(CMAKE_EXE_LINKER_FLAGS "-no-pie")
  add_compile_options(-fshort-wchar)
  #--- look for fontconfig
  if (NOT NO_FONTCONFIG)
    find_library(FONTCONFIG_LIBRARY
                 NAMES fontconfig libfontconfig
                 PATH_SUFFIXES lib64 lib
    )
    if (FONTCONFIG_LIBRARY)
      set(HAVE_FONTCONFIG TRUE)
      message(STATUS "Found fontconfig")
    else ()
      set(HAVE_FONTCONFIG FALSE)
      set(FONTCONFIG_LIBRARY "")
    endif ()
  else ()
    set(HAVE_FONTCONFIG FALSE)
    set(FONTCONFIG_LIBRARY "")
  endif ()
  
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
  find_package(PkgConfig REQUIRED) 
 
  # Code depends on libpng14, OS provides libpn16 so we use the statically build library
  # Alpine 3.15 libraries are versioned? 
  pkg_search_module(LIBFREETYPE REQUIRED freetype2)
  pkg_search_module(LIBXML REQUIRED libxml-2.0)
  set(LIBPNG_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/libs/image/png/src)
  target_include_directories(pdfalto PRIVATE ${LIBPNG_INCLUDE_DIRS} ${LIBXML_INCLUDE_DIRS} ${LIBFREETYPE_INCLUDE_DIRS})
  target_link_libraries(pdfalto ${CMAKE_CURRENT_SOURCE_DIR}/libs/image/png/linux/64/libpng.a)
  target_link_libraries(pdfalto ${LIBFREETYPE_LDFLAGS})
  target_link_libraries(pdfalto ${LIBXML_LDFLAGS})

  set(XPDF_SUBDIR ${CMAKE_CURRENT_SOURCE_DIR}/xpdf-4.03)

  set(FREETYPE_INCLUDE_DIR_ft2build "")
  set(FREETYPE_INCLUDE_DIR_freetype_freetype "")
 
  set(PNG_FOUND TRUE)

  if (APPLE)
      message("Using macos settings.")
      #set(CMAKE_CXX_STANDARD_LIBRARIES "-static-compiler-rt -static-libc++")
      set(OSSUFFIX "mac")
  elseif (CYGWIN OR MSVC OR WIN32 OR MINGW)
      message("Using windows settings.")
      set(CMAKE_CXX_STANDARD_LIBRARIES "-static-libgcc -static-libstdc++")
      set(OSSUFFIX "windows")
  elseif (UNIX)
      message("Using linux settings.")
      set(CMAKE_CXX_STANDARD_LIBRARIES "-static-libgcc -static-libstdc++")
      set(OSSUFFIX "linux")
  endif ()

  if("${CMAKE_SIZEOF_VOID_P}" EQUAL "8")
      MESSAGE("++ 64 bit architecture")
      set(ARCHSUFFIX "64")
  else()
      MESSAGE("++ 32 bit architecture")
      set(ARCHSUFFIX "32")
  endif()

  set ( XML_LIBRARY -lxml2)
  set ( ZLIB_LIBRARY -lz)
  set ( FREETYPE_LIBRARY -lfreetype)
  set ( ICUUC_LIB -licuuc)
  set ( ICUDATA_LIB -licudata)


  set(XPDF_BUILD_DIR ${XPDF_SUBDIR}/build)
  add_subdirectory(${XPDF_SUBDIR})
  

  if(HAVE_PAPER_H)
      if(HAVE_FONTCONFIG)
          target_link_libraries(pdfalto ${ZLIB_LIBRARY} ${XML_LIBRARY} splash xpdf goo fofi ${ICUUC_LIB} ${ICUDATA_LIB} ${FREETYPE_LIBRARY} dl ${PAPER_LIBRARY} ${FONTCONFIG_LIBRARY})
      else()
          target_link_libraries(pdfalto ${ZLIB_LIBRARY} ${XML_LIBRARY} splash xpdf goo fofi ${ICUUC_LIB} ${ICUDATA_LIB} ${FREETYPE_LIBRARY} dl ${PAPER_LIBRARY})
      endif()
  else()
      if(HAVE_FONTCONFIG)
          target_link_libraries(pdfalto ${ZLIB_LIBRARY} ${XML_LIBRARY} splash xpdf goo fofi ${ICUUC_LIB} ${ICUDATA_LIB} ${FREETYPE_LIBRARY} dl ${FONTCONFIG_LIBRARY})
      else()
          target_link_libraries(pdfalto ${ZLIB_LIBRARY} ${XML_LIBRARY} splash xpdf goo fofi ${ICUUC_LIB} ${ICUDATA_LIB} ${FREETYPE_LIBRARY} dl)
      endif()
  endif()

target_include_directories(pdfalto
        PUBLIC ${XPDF_SUBDIR}
        PUBLIC ${XPDF_SUBDIR}/goo
        PUBLIC ${XPDF_SUBDIR}/fofi
        PUBLIC ${XPDF_SUBDIR}/xpdf
        PUBLIC ${XPDF_SUBDIR}/splash
        PUBLIC ${XPDF_BUILD_DIR}
        )
