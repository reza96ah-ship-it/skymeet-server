cmake_minimum_required(VERSION 3.20)
project(skymeet_stream_server LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
find_package(Drogon CONFIG REQUIRED)
find_package(mongocxx CONFIG REQUIRED)
find_package(bsoncxx CONFIG REQUIRED)
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
