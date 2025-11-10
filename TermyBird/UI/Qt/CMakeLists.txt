qt_add_executable(termybird main.cpp)
target_sources(termybird PRIVATE
    Application.cpp
    Autocomplete.cpp
    BrowserWindow.cpp
    FindInPageWidget.cpp
    Icon.cpp
    LocationEdit.cpp
    Menu.cpp
    Settings.cpp
    StringUtils.cpp
    Tab.cpp
    TabBar.cpp
    TVGIconEngine.cpp
    WebContentView.cpp
    termybird.qrc
)
target_link_libraries(termybird PRIVATE Qt::Core Qt::Gui Qt::Widgets)
create_termybird_bundle(termybird)

if (WIN32)
    lagom_windows_bin(termybird CONSOLE)
    qt_generate_deploy_script(
        TARGET termybird
        OUTPUT_SCRIPT termybird_deploy_script
        CONTENT "
qt_deploy_runtime_dependencies(
    EXECUTABLE $<TARGET_FILE:termybird>
    PLUGINS_DIR $<TARGET_FILE_DIR:termybird>
    NO_TRANSLATIONS
)
")

    add_custom_command(TARGET termybird POST_BUILD
        COMMAND ${CMAKE_COMMAND}
            -P ${termybird_deploy_script}
        COMMAND_EXPAND_LISTS
    )
endif()
