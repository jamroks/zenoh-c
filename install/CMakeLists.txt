message(STATUS "zenoh-c install")

declare_cache_var(ZENOHC_INSTALL_STATIC_LIBRARY FALSE BOOL "Install zenoh-c static library")

#
# Installation
# For debug configuration installs libraries with 'd' added to filename and
# package named 'zenohc_debug'
#
status_print(CMAKE_INSTALL_PREFIX)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

function(install_zenohc_lib configurations property_postfix package_name libname_postfix)
    get_target_property(dylib_path zenohc::lib IMPORTED_LOCATION_${property_postfix})
    get_target_property_if_set(implib_path zenohc::lib IMPORTED_IMPLIB_${property_postfix})
    get_target_property(staticlib_path zenohc::static IMPORTED_LOCATION_${property_postfix})
    macro(add_postfix_to_filename dstvar srcvar)
        if(DEFINED ${srcvar})
            get_filename_component(name_we ${${srcvar}} NAME_WE)
            get_filename_component(ext ${${srcvar}} EXT)
            set(${dstvar} ${name_we}${libname_postfix}${ext})
        endif()
    endmacro()
    add_postfix_to_filename(DYLIB dylib_path)
    add_postfix_to_filename(IMPLIB implib_path)
    add_postfix_to_filename(STATICLIB staticlib_path)

    # Install dynamic, import and static library
    install(FILES ${dylib_path} DESTINATION ${CMAKE_INSTALL_LIBDIR} CONFIGURATIONS ${configurations} RENAME ${DYLIB})
    if(DEFINED IMPLIB)
        install(FILES ${implib_path} DESTINATION ${CMAKE_INSTALL_LIBDIR} CONFIGURATIONS ${configurations} RENAME ${IMPLIB})
    endif()
    if(ZENOHC_INSTALL_STATIC_LIBRARY)
        install(FILES ${staticlib_path} DESTINATION ${CMAKE_INSTALL_LIBDIR} CONFIGURATIONS ${configurations} RENAME ${STATICLIB})
    endif()

    set(CMAKE_INSTALL_CMAKEDIR "${CMAKE_INSTALL_LIBDIR}/cmake/${package_name}")

    # Generate <Package>Config.cmake
    configure_package_config_file(
        "PackageConfig.cmake.in"
        "${CMAKE_CURRENT_BINARY_DIR}/${package_name}Config.cmake"
        INSTALL_DESTINATION "${CMAKE_INSTALL_CMAKEDIR}")

    # Generate <Package>Version.cmake
    write_basic_package_version_file(
        "${CMAKE_CURRENT_BINARY_DIR}/${package_name}ConfigVersion.cmake"
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion)

    install(
        FILES "${CMAKE_CURRENT_BINARY_DIR}/${package_name}Config.cmake"
              "${CMAKE_CURRENT_BINARY_DIR}/${package_name}ConfigVersion.cmake"
        DESTINATION "${CMAKE_INSTALL_CMAKEDIR}"
        CONFIGURATIONS ${configurations}
        COMPONENT dev)

    if(APPLE OR UNIX)
        set(LIBNAME_POSTFIX ${libname_postfix})
        configure_file(${CMAKE_CURRENT_SOURCE_DIR}/zenohc.pc.in ${CMAKE_CURRENT_BINARY_DIR}/${package_name}.pc @ONLY)
        install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${package_name}.pc 
            DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig" 
            CONFIGURATIONS ${configurations}
            OPTIONAL)
    endif()
endfunction()


#
# Install headers
#
get_target_property(include_dirs zenohc::lib INTERFACE_INCLUDE_DIRECTORIES)
foreach(dir ${include_dirs})
    install(DIRECTORY "${dir}/" DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
endforeach()

#
# Install lib files 
#
install_zenohc_lib("Release;RelWithDebInfo;MinSizeRel" "RELEASE" zenohc "")
install_zenohc_lib("Debug" "DEBUG" zenohc_debug "d")