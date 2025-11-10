include(${LADYBIRD_SOURCE_DIR}/UI/cmake/ResourceFiles.cmake)

function(create_termybird_bundle target_name)
    set_target_properties(${target_name} PROPERTIES
    OUTPUT_NAME "termybird"
        MACOSX_BUNDLE_GUI_IDENTIFIER org.termybird.TermyBird
        MACOSX_BUNDLE_BUNDLE_VERSION ${PROJECT_VERSION}
        MACOSX_BUNDLE_SHORT_VERSION_STRING ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}
        MACOSX_BUNDLE_INFO_PLIST "${LADYBIRD_SOURCE_DIR}/UI/Info.plist"
        MACOSX_BUNDLE TRUE
        WIN32_EXECUTABLE TRUE
        XCODE_ATTRIBUTE_PRODUCT_BUNDLE_IDENTIFIER org.termybird.TermyBird
    )

    set(termybird_should_copy_resources TRUE)

    if (APPLE)
        set(bundle_dir "$<TARGET_BUNDLE_DIR:${target_name}>")
        add_custom_command(TARGET ${target_name} POST_BUILD
            COMMAND "${CMAKE_COMMAND}" -E make_directory "${bundle_dir}/Contents/Resources"
            COMMAND "iconutil" --convert icns "${LADYBIRD_SOURCE_DIR}/UI/Icons/macos/app_icon.iconset" --output "${bundle_dir}/Contents/Resources/app_icon.icns"
        )

        add_custom_command(TARGET ${target_name} POST_BUILD
            COMMAND "${CMAKE_COMMAND}" -E create_symlink "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}" "${bundle_dir}/Contents/lib"
        )

        if (NOT CMAKE_BUILD_TYPE MATCHES "Release|RelWithDebInfo" AND "arm64" IN_LIST CMAKE_OSX_ARCHITECTURES)
            add_custom_command(TARGET ${target_name} POST_BUILD
                COMMAND codesign -s - -v -f --entitlements "${LADYBIRD_SOURCE_DIR}/Meta/debug.plist" "${bundle_dir}"
            )
        else()
            add_custom_target(apply-termybird-debug-entitlements
                COMMAND codesign -s - -v -f --entitlements "${LADYBIRD_SOURCE_DIR}/Meta/debug.plist" "${bundle_dir}"
                USES_TERMINAL
            )
        endif()

        set(resource_base_dir "$<TARGET_BUNDLE_DIR:${target_name}>/Contents/Resources")
    elseif (TARGET ladybird_build_resource_files)
        add_dependencies(${target_name} ladybird_build_resource_files)
        set(termybird_should_copy_resources FALSE)
    else()
        set(resource_base_dir "${CMAKE_BINARY_DIR}/${IN_BUILD_PREFIX}${CMAKE_INSTALL_DATADIR}/TermyBird")
    endif()

    if (termybird_should_copy_resources)
        copy_resources_to_build(${resource_base_dir} ${target_name})
    endif()
endfunction()

function(set_termybird_helper_process_properties)
    set(targets ${ARGV})
    if (APPLE OR WIN32)
        set_target_properties(${targets} PROPERTIES RUNTIME_OUTPUT_DIRECTORY "$<TARGET_FILE_DIR:termybird>")
    else()
        set_target_properties(${targets} PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/${IN_BUILD_PREFIX}${CMAKE_INSTALL_LIBEXECDIR}")
    endif()
endfunction()

if (NOT ENABLE_QT)
    message(FATAL_ERROR "TermyBird requires Qt to be enabled")
endif()

add_subdirectory(UI/Qt)

set(TERMYBIRD_LIBS AK LibCore LibFileSystem LibGfx LibImageDecoderClient LibIPC LibJS LibMain LibWeb LibWebView LibWebViewPlatform LibRequests LibURL)

if (TARGET termybird_impl)
    set(TERMYBIRD_TARGET termybird_impl)
else()
    set(TERMYBIRD_TARGET termybird)
endif()

target_link_libraries(${TERMYBIRD_TARGET} PRIVATE ${TERMYBIRD_LIBS})
target_link_libraries(${TERMYBIRD_TARGET} PRIVATE OpenSSL::Crypto OpenSSL::SSL)

target_include_directories(${TERMYBIRD_TARGET} PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
target_include_directories(${TERMYBIRD_TARGET} BEFORE PRIVATE ${LADYBIRD_SOURCE_DIR}/TermyBird)
target_include_directories(${TERMYBIRD_TARGET} PRIVATE ${LADYBIRD_SOURCE_DIR})
target_include_directories(${TERMYBIRD_TARGET} PRIVATE ${LADYBIRD_SOURCE_DIR}/Services)

if (WIN32)
    target_link_libraries(${TERMYBIRD_TARGET} LibDevTools)
endif()

set(termybird_helper_processes ImageDecoder RequestServer WebContent WebWorker)

add_dependencies(termybird ${termybird_helper_processes})
if (NOT WIN32)
    add_dependencies(WebDriver termybird)
endif()

set_termybird_helper_process_properties(${termybird_helper_processes})
if (APPLE)
    set_termybird_helper_process_properties(WebDriver)
endif()
