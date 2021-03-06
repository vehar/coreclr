
# Set the RPATH of mscordbi so that it can find dependencies without needing to set LD_LIBRARY_PATH
# For more information: http://www.cmake.org/Wiki/CMake_RPATH_handling.
if(CORECLR_SET_RPATH)
  set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
  if(CLR_CMAKE_PLATFORM_DARWIN)
    set(CMAKE_INSTALL_RPATH "@loader_path")
  else()
    set(CMAKE_INSTALL_RPATH "\$ORIGIN")
  endif(CLR_CMAKE_PLATFORM_DARWIN)
endif(CORECLR_SET_RPATH)

set(MSCORDBI_SOURCES
  mscordbi.cpp
)  

if(WIN32)
    add_precompiled_header(stdafx.h stdafx.cpp MSCORDBI_SOURCES)
    
    add_definitions(-DFX_VER_INTERNALNAME_STR=mscordbi.dll)

    list(APPEND MSCORDBI_SOURCES
        Native.rc
    )

    set(DEF_SOURCES
      mscordbi.src
    )

    convert_to_absolute_path(DEF_SOURCES ${DEF_SOURCES})

    preprocess_def_file(${DEF_SOURCES} ${CMAKE_CURRENT_BINARY_DIR}/mscordbi.def)

    list(APPEND MSCORDBI_SOURCES ${CMAKE_CURRENT_BINARY_DIR}/mscordbi.def)
else(WIN32)
    set(DEF_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/mscordbi_unixexports.src)
    set(EXPORTS_FILE ${CMAKE_CURRENT_BINARY_DIR}/mscordbi.exports)
    generate_exports_file(${DEF_SOURCES} ${EXPORTS_FILE})
endif(WIN32)

if(CLR_CMAKE_PLATFORM_LINUX OR CLR_CMAKE_PLATFORM_FREEBSD OR CLR_CMAKE_PLATFORM_NETBSD)
    # This option is necessary to ensure that the overloaded new/delete operators defined inside
    # of the utilcode will be used instead of the standard library delete operator.
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Xlinker -Bsymbolic -Xlinker -Bsymbolic-functions")    

    # Add linker exports file option
    set(EXPORTS_LINKER_OPTION -Wl,--version-script=${EXPORTS_FILE})
endif(CLR_CMAKE_PLATFORM_LINUX OR CLR_CMAKE_PLATFORM_FREEBSD OR CLR_CMAKE_PLATFORM_NETBSD)

if(CLR_CMAKE_PLATFORM_DARWIN)
    # Add linker exports file option
    set(EXPORTS_LINKER_OPTION -Wl,-exported_symbols_list,${EXPORTS_FILE})
endif(CLR_CMAKE_PLATFORM_DARWIN)

add_library_clr(mscordbi SHARED ${MSCORDBI_SOURCES})

if(CLR_CMAKE_PLATFORM_UNIX)
    add_custom_target(mscordbi_exports DEPENDS ${EXPORTS_FILE})
    add_dependencies(mscordbi mscordbi_exports)

    set_property(TARGET mscordbi APPEND_STRING PROPERTY LINK_FLAGS ${EXPORTS_LINKER_OPTION})
    set_property(TARGET mscordbi APPEND_STRING PROPERTY LINK_DEPENDS ${EXPORTS_FILE})
endif(CLR_CMAKE_PLATFORM_UNIX)

set(COREDBI_LIBRARIES
  debug-pal
  cordbdi
  utilcodestaticnohost
  ildbsymlib
  mdcompiler-dbi
  mdruntime-dbi
  mdruntimerw-dbi
  mddatasource_dbi
  corguids
) 

if(WIN32)
    list(APPEND COREDBI_LIBRARIES
        ipcmanager-staticcrt
        mdhotdata-staticcrt
        mdwinmd_dbi
        kernel32.lib
        advapi32.lib
        ole32.lib
        oleaut32.lib
        uuid.lib
        user32.lib
        version.lib
        ${STATIC_MT_CRT_LIB}
        ${STATIC_MT_VCRT_LIB}
    )

    target_link_libraries(mscordbi ${COREDBI_LIBRARIES})

elseif(CLR_CMAKE_PLATFORM_UNIX)

    list(APPEND COREDBI_LIBRARIES
        mdhotdata_full
        palrt
        # share the PAL in the dac module
        mscordaccore
    )

    add_dependencies(mscordbi mscordaccore)

    # COREDBI_LIBRARIES is mentioned twice because ld is one pass linker and will not find symbols
    # if they are defined after they are used. Having all libs twice makes sure that ld will actually
    # find all symbols.
    target_link_libraries(mscordbi ${COREDBI_LIBRARIES} ${COREDBI_LIBRARIES})

endif(WIN32)

# add the install targets
install_clr(mscordbi)