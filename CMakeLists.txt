cmake_minimum_required(VERSION 3.20)
project(skymeet_stream_server LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

option(SKYMEET_CLONE_MISSING_DEPS "Clone missing dependencies from GitHub into third_party/" OFF)
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

    message(FATAL_ERROR
        "Missing packages were cloned to ${SKYMEET_THIRD_PARTY_DIR}, but they must still be built/installed.\n"
        "Re-run CMake with CMAKE_PREFIX_PATH or *_DIR pointing to installed dependency locations."
    )
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

if(SKYMEET_MISSING_DEPS)
    list(JOIN SKYMEET_MISSING_DEPS ", " SKYMEET_MISSING_DEPS_TEXT)
    message(FATAL_ERROR
        "Missing required dependencies: ${SKYMEET_MISSING_DEPS_TEXT}.\n"
        "Install development packages and set CMAKE_PREFIX_PATH (or *_DIR vars) so CMake can find them.\n"
        "To clone missing sources from GitHub, configure with -DSKYMEET_CLONE_MISSING_DEPS=ON.\n"
        "See README.md -> Build dependencies for examples."
    )
endif()

file(GLOB_RECURSE SKYMEET_SOURCES CONFIGURE_DEPENDS src/*.cc)
add_executable(skymeet_stream_server ${SKYMEET_SOURCES})
target_include_directories(skymeet_stream_server PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src)
target_link_libraries(skymeet_stream_server PRIVATE Drogon::Drogon mongo::mongocxx_shared mongo::bsoncxx_shared)

# ------------------------------------------------------------
# Runtime folders and config copy
# ------------------------------------------------------------
set(SKYMEET_RUNTIME_DIR ${CMAKE_BINARY_DIR})
set(SKYMEET_CONFIG_SOURCE_DIR ${CMAKE_SOURCE_DIR}/config)
set(SKYMEET_CONFIG_OUTPUT_DIR ${SKYMEET_RUNTIME_DIR}/config)
set(SKYMEET_LOGS_OUTPUT_DIR ${SKYMEET_RUNTIME_DIR}/logs)

file(MAKE_DIRECTORY ${SKYMEET_CONFIG_OUTPUT_DIR})
file(MAKE_DIRECTORY ${SKYMEET_LOGS_OUTPUT_DIR})

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
