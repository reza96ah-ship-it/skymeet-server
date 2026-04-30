# SkyMeet Server

Drogon C++ backend for SkyMeet stream sessions.

## Features

- Drogon HTTP API
- MongoDB persistence
- ZLMediaKit integration
- Auth/login module
- Stream sessions
- ZLMediaKit hook endpoints
- WebSocket event hub
- CMake runtime config copy and logs folder creation

## Build dependencies

Install the required C++ SDKs before configuring CMake:

- Drogon (must provide `DrogonConfig.cmake`)
- MongoDB C++ driver `mongocxx` and `bsoncxx` (must provide `mongocxxConfig.cmake` and `bsoncxxConfig.cmake`)

If packages are installed in a non-system prefix, point CMake to them:

```bash
cmake -S . -B build \
  -DCMAKE_PREFIX_PATH="/path/to/drogon;/path/to/mongo-cxx-driver"
```

You can also set `Drogon_DIR`, `mongocxx_DIR`, and `bsoncxx_DIR` individually.

To clone missing dependency source code directly from GitHub via CMake:

```bash
cmake -S . -B build -DSKYMEET_CLONE_MISSING_DEPS=ON
```

This clones repositories into `third_party/` (or `SKYMEET_THIRD_PARTY_DIR`) and then stops with instructions, because Drogon and mongo-cxx-driver still need to be built and installed before this project can link against them.

## Build

```bash
mkdir build
cd build
cmake ..
cmake --build . -j
./skymeet_stream_server config/config.json
```

## Runtime folders

The build creates:

```text
build/config/
build/logs/
```

