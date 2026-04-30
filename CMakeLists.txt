cmake_minimum_required(VERSION 3.20)
project(skymeet_stream_server LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

option(SKYMEET_CLONE_MISSING_DEPS "Clone missing dependencies from GitHub into third_party/" OFF)
option(SKYMEET_ALLOW_STUB_BUILD "Build a stub binary when required SDKs are missing" ON)
set(SKYMEET_THIRD_PARTY_DIR "${CMAKE_SOURCE_DIR}/third_party" CACHE PATH "Directory used for cloned third-party dependencies")

function(skymeet_clone_if_missing dep_name repo_url output_dir)
    if(EXISTS "${output_dir}/.git")
        message(STATUS "${dep_name} already cloned at ${output_dir}")
        return()
    endif()

    file(MAKE_DIRECTORY "${SKYMEET_THIRD_PARTY_DIR}")
    message(STATUS "Cloning ${dep_name} from ${repo_url} into ${output_dir}")
    execute_process(
        COMMAND git clone --depth 1 "${repo_url}" "${output_dir}"
        RESULT_VARIABLE SKYMEET_CLONE_RESULT
        OUTPUT_VARIABLE SKYMEET_CLONE_STDOUT
        ERROR_VARIABLE SKYMEET_CLONE_STDERR
    )

    if(NOT SKYMEET_CLONE_RESULT EQUAL 0)
        message(FATAL_ERROR
            "Failed to clone ${dep_name} from ${repo_url}.\n"
            "stdout:\n${SKYMEET_CLONE_STDOUT}\n"
            "stderr:\n${SKYMEET_CLONE_STDERR}"
        )
    endif()
endfunction()

# ------------------------------------------------------------
# Dependency discovery
# ------------------------------------------------------------
find_package(Drogon CONFIG QUIET)
find_package(mongocxx CONFIG QUIET)
find_package(bsoncxx CONFIG QUIET)

if(SKYMEET_CLONE_MISSING_DEPS)
    if(NOT Drogon_FOUND)
        skymeet_clone_if_missing("Drogon" "https://github.com/drogonframework/drogon.git" "${SKYMEET_THIRD_PARTY_DIR}/drogon")
    endif()
    if(NOT mongocxx_FOUND OR NOT bsoncxx_FOUND)
        skymeet_clone_if_missing("mongo-cxx-driver" "https://github.com/mongodb/mongo-cxx-driver.git" "${SKYMEET_THIRD_PARTY_DIR}/mongo-cxx-driver")
    endif()
endif()

set(SKYMEET_MISSING_DEPS "")
if(NOT Drogon_FOUND)
    list(APPEND SKYMEET_MISSING_DEPS "Drogon")
endif()
if(NOT mongocxx_FOUND)
    list(APPEND SKYMEET_MISSING_DEPS "mongocxx")
endif()
if(NOT bsoncxx_FOUND)
    list(APPEND SKYMEET_MISSING_DEPS "bsoncxx")
endif()

set(SKYMEET_USING_STUB_BUILD OFF)
if(SKYMEET_MISSING_DEPS)
    list(JOIN SKYMEET_MISSING_DEPS ", " SKYMEET_MISSING_DEPS_TEXT)
    if(SKYMEET_ALLOW_STUB_BUILD)
        set(SKYMEET_USING_STUB_BUILD ON)
        message(WARNING
            "Missing required dependencies: ${SKYMEET_MISSING_DEPS_TEXT}.\n"
            "Building stub binary because SKYMEET_ALLOW_STUB_BUILD=ON.\n"
            "Install SDKs and set CMAKE_PREFIX_PATH or *_DIR vars for full server build."
        )
    else()
        message(FATAL_ERROR
            "Missing required dependencies: ${SKYMEET_MISSING_DEPS_TEXT}.\n"
            "Install development packages and set CMAKE_PREFIX_PATH (or *_DIR vars) so CMake can find them.\n"
            "To clone missing sources from GitHub, configure with -DSKYMEET_CLONE_MISSING_DEPS=ON.\n"
            "See README.md -> Build dependencies for examples."
        )
    endif()
endif()

if(SKYMEET_USING_STUB_BUILD)
    set(SKYMEET_STUB_SOURCE "${CMAKE_BINARY_DIR}/skymeet_stub_main.cc")
    file(WRITE "${SKYMEET_STUB_SOURCE}" "#include <iostream>\nint main(){std::cout << \"skymeet_stream_server stub build: missing required SDKs\\n\"; return 0;}\n")
    add_executable(skymeet_stream_server "${SKYMEET_STUB_SOURCE}")
else()
    file(GLOB_RECURSE SKYMEET_SOURCES CONFIGURE_DEPENDS src/*.cc)
    add_executable(skymeet_stream_server ${SKYMEET_SOURCES})
    target_include_directories(skymeet_stream_server PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src)
    target_link_libraries(skymeet_stream_server PRIVATE Drogon::Drogon mongo::mongocxx_shared mongo::bsoncxx_shared)
endif()

# ------------------------------------------------------------
# Runtime folders and config copy
# ------------------------------------------------------------
set(SKYMEET_RUNTIME_DIR ${CMAKE_BINARY_DIR})
set(SKYMEET_CONFIG_SOURCE_DIR ${CMAKE_SOURCE_DIR}/config)
set(SKYMEET_CONFIG_OUTPUT_DIR ${SKYMEET_RUNTIME_DIR}/config)
set(SKYMEET_LOGS_OUTPUT_DIR ${SKYMEET_RUNTIME_DIR}/logs)

file(MAKE_DIRECTORY ${SKYMEET_CONFIG_OUTPUT_DIR})
file(MAKE_DIRECTORY ${SKYMEET_LOGS_OUTPUT_DIR})

if(EXISTS ${SKYMEET_CONFIG_SOURCE_DIR})
    file(COPY ${SKYMEET_CONFIG_SOURCE_DIR}
         DESTINATION ${SKYMEET_RUNTIME_DIR})

    add_custom_command(
        TARGET skymeet_stream_server
        POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E make_directory ${SKYMEET_CONFIG_OUTPUT_DIR}
        COMMAND ${CMAKE_COMMAND} -E make_directory ${SKYMEET_LOGS_OUTPUT_DIR}
        COMMAND ${CMAKE_COMMAND} -E copy_directory
                ${SKYMEET_CONFIG_SOURCE_DIR}
                ${SKYMEET_CONFIG_OUTPUT_DIR}
        COMMENT "Copying config folder and creating logs folder"
    )
else()
    message(WARNING "Config directory not found at ${SKYMEET_CONFIG_SOURCE_DIR}; skipping config copy.")
    add_custom_command(
        TARGET skymeet_stream_server
        POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E make_directory ${SKYMEET_CONFIG_OUTPUT_DIR}
        COMMAND ${CMAKE_COMMAND} -E make_directory ${SKYMEET_LOGS_OUTPUT_DIR}
        COMMENT "Creating runtime folders"
    )
endif()
