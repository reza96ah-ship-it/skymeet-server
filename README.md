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

