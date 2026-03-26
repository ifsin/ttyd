# AGENTS.md

## Project Overview

**ttyd** is a command-line tool that shares a terminal session over the web. It has two components:

- **Backend** (`src/`): C99 server using libwebsockets + libuv for WebSocket/HTTP serving and PTY management
- **Frontend** (`html/`): TypeScript/Preact web UI using xterm.js for terminal rendering, compiled into a C header (`src/html.h`) and embedded in the binary

## Essential Commands

### Backend (C, CMake)

```bash
# Build (requires: libwebsockets (>=3.2.0, with libuv), libuv, json-c, zlib)
mkdir build && cd build
cmake .. && make

# Build on Windows (MSVC, via vcpkg)
cmake -S . -B build -G "Visual Studio 17 2022" -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_TOOLCHAIN_FILE="$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake" \
  -DVCPKG_TARGET_TRIPLET=x64-windows-static
cmake --build build --config Release
```

### Frontend (TypeScript, Yarn 3)

```bash
cd html
corepack enable && corepack prepare yarn@stable --activate
yarn install

# Development server (hot reload)
yarn start

# Production build
yarn build                  # webpack + gulp -> generates src/html.h

# Lint/check
yarn check                  # gts check (Google TypeScript style)
yarn fix                    # gts fix
```

### Cross-Compilation

```bash
# Build for a specific architecture (Linux)
BUILD_TARGET=aarch64 ./scripts/cross-build.sh

# Supported targets: i686, x86_64, arm, armhf, armv7l, aarch64,
#   mips, mipsel, mips64, mips64el, ppc64, ppc64le, s390x, win32
```

### Docker

```bash
docker build -t ttyd .
docker build -t ttyd-alpine -f Dockerfile.alpine .
```

### Snap

```bash
snapcraft
```

## Code Organization

```
src/
  server.c      - Main entry point, CLI parsing, libuv/lws setup
  server.h      - Core data structures (server, pss_tty, pss_http, endpoints)
  protocol.c    - WebSocket callback: TTY session handling, auth, process spawning
  http.c        - HTTP callback: serves embedded HTML, auth, token endpoint
  pty.c         - PTY process management (Unix: forkpty; Windows: ConPTY)
  pty.h         - PTY types and API
  utils.c       - Utilities (xmalloc, xrealloc, signal handling, open_uri)
  utils.h       - Utility declarations
  compat.h      - MSVC compatibility shims (strcasecmp, S_ISDIR, etc.)
  html.h        - AUTO-GENERATED: gzipped HTML embedded as C byte array

html/
  src/
    index.tsx               - Entry point, renders <App>
    components/
      app.tsx               - Root component, configures WS URL and terminal options
      terminal/index.tsx    - Terminal wrapper component
      terminal/xterm/       - xterm.js integration, WebSocket protocol, addons
      modal/index.tsx       - File picker modal
    style/index.scss        - Styles
  gulpfile.js               - Generates html.h from built HTML (gzip -> C array)
  webpack.config.js         - Webpack build config
```

## Architecture Details

### Client-Server Protocol (WebSocket binary messages)

Messages are prefixed with a single-char command type:

| Direction | Char | Meaning |
|-----------|------|---------|
| Client->Server | `'0'` | Terminal input |
| Client->Server | `'1'` | Resize terminal (JSON: `{"columns":N,"rows":N}`) |
| Client->Server | `'2'` | Pause output |
| Client->Server | `'3'` | Resume output |
| Client->Server | `'{'` | JSON data (initial auth + window size) |
| Server->Client | `'0'` | Terminal output |
| Server->Client | `'1'` | Set window title |
| Server->Client | `'2'` | Set preferences (JSON) |

### Authentication

Two modes:
1. **Basic auth**: `-c username:password` -- base64 encoded, checked via HTTP header and WS auth token
2. **Proxy auth header**: `-H header-name` -- delegates to reverse proxy, reads custom header

### HTML Embedding Pipeline

1. `yarn build` runs webpack (bundles TS/SCSS) then gulp
2. Gulp inlines CSS/JS into single HTML, gzips it, and writes `src/html.h` as a C byte array
3. `http.c` serves the embedded HTML, decompressing at runtime if client doesn't accept gzip

## Code Style and Conventions

### C Code

- **Standard**: C99 (set via `CMAKE_C_STANDARD 99`)
- **Formatting**: Google style via `.clang-format` -- 2-space indent, 120 column limit, no tabs
- **Memory**: `xmalloc()` / `xrealloc()` wrappers that abort on failure (in `utils.c`)
- **Platform guards**: `#ifdef _WIN32` for Windows (ConPTY), `#ifdef __APPLE__` for macOS
- **Logging**: libwebsockets logging macros (`lwsl_notice`, `lwsl_warn`, `lwsl_err`)
- **No test suite**: there are no unit tests in the repository

### TypeScript/Frontend

- **Framework**: Preact (not React)
- **Linting**: Google TypeScript Style (`gts`) via `yarn check`
- **Package manager**: Yarn 3 (Berry) with `.yarnrc.yml`
- **Dependencies**: xterm.js v5 with addons (canvas, clipboard, fit, image, unicode11, web-links, webgl), zmodem.js (patched), trzsz

## Important Gotchas

1. **Frontend must be built before C backend**: `html/gulpfile.js` generates `src/html.h` which is included by `src/http.c`. Building C without this file will fail.

2. **libwebsockets must be compiled with `-DLWS_WITH_LIBUV=ON`**: CMake checks for `LWS_WITH_LIBUV` and fails if absent (`CMakeLists.txt:79-81`).

3. **Version must match git tag for releases**: The release workflow (`release.yml:17-23`) verifies that the version in `CMakeLists.txt` matches the git tag.

4. **zmodem.js is patched**: There's a Yarn patch at `html/.yarn/patches/zmodem.js-npm-0.1.10-e5537fa2ed.patch` applied via `resolutions` in `package.json`.

5. **`html.h` is auto-generated**: Never edit `src/html.h` directly -- it's produced by the gulp pipeline. Edit frontend source in `html/src/` instead.

6. **Windows ConPTY requires Windows 10 1809+**: The ConPTY API is dynamically loaded at runtime (`pty.c:conpty_init()`), and the application exits with an error on older Windows versions.

7. **`environ` on macOS**: Uses `_NSGetEnviron()` from `<crt_externs.h>` instead of the `environ` global (`pty.c:23-28`).

8. **Cross-compilation uses musl toolchains**: The `scripts/cross-build.sh` downloads musl-based cross-compilers from `github.com/tsl0922/musl-toolchains` and statically links all dependencies.

## CI Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `backend.yml` | Push/PR to `src/`, `CMakeLists.txt`, `scripts/` | Cross-build for 13 targets + MSVC build |
| `frontend.yml` | Push/PR to `html/` | Lint and build frontend |
| `docker.yml` | Push to main or tags | Build and push Docker images (amd64, armv7, arm64, s390x) |
| `release.yml` | Tags | Build all targets, create GitHub draft release with SHA256SUMS |

## Dependencies (C)

| Library | Purpose | Minimum Version |
|---------|---------|-----------------|
| libwebsockets | HTTP/WebSocket server | 3.2.0 (must have libuv support) |
| libuv | Async I/O event loop | -- |
| json-c | JSON parsing | -- |
| zlib | HTML compression | -- |
| OpenSSL (optional) | TLS/SSL support | -- |
