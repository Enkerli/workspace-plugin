cmake_minimum_required(VERSION 3.22)
project(Workspace VERSION 0.1.0)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED YES)

# enkerli-juce is a git submodule — a plain `git clone` leaves it empty and
# the include below then fails with a cryptic "Unknown CMake command".
# Initialize it automatically so fresh clones just work.
if(NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/enkerli-juce/cmake/EnkerliPlugin.cmake")
    find_package(Git QUIET)
    if(GIT_FOUND AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/.git")
        message(STATUS "enkerli-juce submodule is empty — running git submodule update --init")
        execute_process(COMMAND "${GIT_EXECUTABLE}" submodule update --init enkerli-juce
                        WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
                        RESULT_VARIABLE _submod_rc)
    endif()
    if(NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/enkerli-juce/cmake/EnkerliPlugin.cmake")
        message(FATAL_ERROR "enkerli-juce submodule missing. Run: git submodule update --init")
    endif()
endif()
include("${CMAKE_CURRENT_SOURCE_DIR}/enkerli-juce/cmake/EnkerliPlugin.cmake")
enkerli_resolve_juce()

# Per-machine signing override (gitignored) — see signing.local.cmake.example.
include("${CMAKE_CURRENT_SOURCE_DIR}/signing.local.cmake" OPTIONAL)

# ── Plugin (aumi archetype — the suite's known-good MIDI-effect settings) ────
# Identity is forever: Wksp / com.enkerli.workspace (plugin-codes-are-forever).
enkerli_add_midi_effect_plugin(Workspace
    PRODUCT_NAME "Suite Workspace"
    PLUGIN_CODE  Wksp
    BUNDLE_ID    "com.enkerli.workspace"
    VERSION      "${PROJECT_VERSION}"
    DESCRIPTION  "Suite modules on one bus, in a DAW: GloriArp grooves out as MIDI, hardware MIDI in through the bindings"
    ICON_BIG     "${CMAKE_SOURCE_DIR}/Assets/icon_1024.png"
)

target_sources(Workspace PRIVATE Source/Plugin.cpp)
if(APPLE)
    # Native save (share sheet) + open (document picker) paths for the
    # WebView UI (ObjC++; both compile empty on macOS where FileChooser/
    # NSOpenPanel are used instead).
    target_sources(Workspace PRIVATE
        enkerli-juce/src/FileExport.mm
        enkerli-juce/src/FileImport.mm)
endif()

# ── WebUI: built from the music-suite monorepo at configure/build time ───────
# The UI SOURCE lives in music-suite (apps/workspace) — single source shared
# with the browser webapp; only the C++ shell lives here (the Serpe pattern).
# Default to a sibling checkout; override per machine in webui.local.cmake.
# The monorepo must be `npm install`ed so esbuild + deps resolve from its
# hoisted node_modules.
set(WORKSPACE_WEBUI_DIR "${CMAKE_SOURCE_DIR}/../music-suite/apps/workspace"
    CACHE PATH "Path to the Workspace WebUI source (music-suite/apps/workspace)")
include("${CMAKE_CURRENT_SOURCE_DIR}/webui.local.cmake" OPTIONAL)
# The default above assumes music-suite is a SIBLING checkout. Before failing,
# probe the other supported layouts: this repo nested INSIDE the monorepo
# (…/music-suite/Workspace), and a MUSIC_SUITE environment variable pointing at
# the monorepo checkout. webui.local.cmake still wins when present.
if(NOT EXISTS "${WORKSPACE_WEBUI_DIR}/main.js")
    foreach(_webui_cand "${CMAKE_SOURCE_DIR}/../apps/workspace" "$ENV{MUSIC_SUITE}/apps/workspace")
        if(EXISTS "${_webui_cand}/main.js")
            set(WORKSPACE_WEBUI_DIR "${_webui_cand}" CACHE PATH
                "Path to the Workspace WebUI source (music-suite/apps/workspace)" FORCE)
            message(STATUS "Workspace WebUI: found at ${_webui_cand}")
            break()
        endif()
    endforeach()
endif()
set(WEBUI_DIR   "${WORKSPACE_WEBUI_DIR}")
set(WEBUI_BUILD "${CMAKE_CURRENT_BINARY_DIR}/webui")

if(NOT EXISTS "${WEBUI_DIR}/main.js")
    message(FATAL_ERROR
        "Workspace WebUI source not found at ${WEBUI_DIR}. It lives in the "
        "music-suite monorepo (apps/workspace). Check out music-suite next to "
        "this repo, or set WORKSPACE_WEBUI_DIR in webui.local.cmake, or export MUSIC_SUITE=/path/to/music-suite. Layouts probed automatically: sibling checkout (../music-suite) and this repo nested inside the monorepo (../apps/workspace).")
endif()

find_program(NPXCMD npx)
if(NPXCMD)
    file(GLOB WEBUI_SRC
        "${WEBUI_DIR}/*.js" "${WEBUI_DIR}/*.css" "${WEBUI_DIR}/index.html")
    add_custom_command(
        OUTPUT  "${WEBUI_BUILD}/bundle.js"
        COMMAND "${CMAKE_COMMAND}" -E make_directory "${WEBUI_BUILD}"
        COMMAND "${NPXCMD}" esbuild "${WEBUI_DIR}/main.js"
                    --bundle --conditions=source
                    --loader:.css=text --loader:.svg=dataurl
                    --loader:.txt=text --loader:.wasm=binary
                    "--define:process.env.NODE_ENV=\"production\"" --minify
                    "--outfile=${WEBUI_BUILD}/bundle.js"
        DEPENDS ${WEBUI_SRC}
        WORKING_DIRECTORY "${WEBUI_DIR}"
        COMMENT "Bundling Workspace WebUI with esbuild"
        VERBATIM)
    add_custom_target(WorkspaceWebUIBundle DEPENDS "${WEBUI_BUILD}/bundle.js")
else()
    message(WARNING "npx not found — Workspace WebUI bundle.js must be pre-built at ${WEBUI_BUILD}/bundle.js")
endif()

set_source_files_properties("${WEBUI_BUILD}/bundle.js" PROPERTIES GENERATED TRUE)
juce_add_binary_data(WorkspaceWebUI
    HEADER_NAME BinaryDataWebUI.h
    NAMESPACE   BinaryData
    SOURCES
        "${WEBUI_DIR}/index.html"
        "${WEBUI_BUILD}/bundle.js")
if(NPXCMD)
    add_dependencies(WorkspaceWebUI WorkspaceWebUIBundle)
endif()

target_link_libraries(Workspace PRIVATE
    WorkspaceWebUI
    juce::juce_audio_utils
    juce::juce_gui_extra
    PUBLIC
    juce::juce_recommended_config_flags
    juce::juce_recommended_warning_flags)
