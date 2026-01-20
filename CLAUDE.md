# CLAUDE.md - AI Assistant Development Guide

## Project Overview

**jsmpeg-vnc** is a low-latency, high-framerate screen sharing server and client for Windows. It captures windows or the desktop, encodes them using MPEG1 video, and streams them via WebSocket to any modern browser. The client is viewable in real-time with minimal latency.

### Key Features
- Real-time screen/window capture and streaming
- MPEG1 video encoding with ffmpeg
- WebSocket-based communication
- Browser-based client with keyboard and mouse input support
- Mouse lock support for games
- Configurable bitrate, framerate, output size, and crop area
- Mobile touch controls for basic keyboard keys

### Technology Stack
- **Backend**: C/C++ (Windows-specific)
- **Video Encoding**: ffmpeg (MPEG1 codec)
- **WebSocket Server**: libwebsockets
- **Client**: JavaScript (jsmpeg decoder)
- **Build System**: Visual Studio (MSVC)

---

## Repository Structure

```
jsmpeg-vnc/
├── client/                  # Browser-based client application
│   ├── index.html          # Main HTML page
│   ├── jsmpg.js            # Modified jsmpeg decoder (handles custom frame headers)
│   └── jsmpg-vnc.js        # Client-side input handling and WebSocket connection
│
├── source/                  # C/C++ server source code
│   ├── jsmpeg-vnc.c        # Main entry point, command-line argument parsing
│   ├── app.c/h             # Application logic, frame loop, input handling
│   ├── encoder.c/h         # ffmpeg-based MPEG1 video encoding
│   ├── grabber.c/h         # Windows screen/window capture (GDI)
│   ├── server.c/h          # WebSocket server wrapper around libwebsockets
│   ├── timer.c/h           # High-precision timing utilities
│   ├── ffmpeg/             # ffmpeg libraries (headers and precompiled libs)
│   └── libwebsocket/       # libwebsockets library
│
├── vs/                      # Visual Studio project files
│   ├── jsmpeg-vnc.sln      # Solution file
│   ├── jsmpeg-vnc.vcxproj  # Project file
│   └── jsmpeg-vnc.vcxproj.filters
│
├── .gitignore              # Git ignore rules (VS, build artifacts)
└── README.md               # User-facing documentation

```

---

## Architecture Overview

### Data Flow

```
Windows GDI → Screen Grabber → MPEG1 Encoder → WebSocket Server → Browser Client
    ↑                                                                      ↓
    └────────────────────── Input Events (Keyboard/Mouse) ───────────────┘
```

### Component Breakdown

#### 1. **Main Entry Point** (`jsmpeg-vnc.c`)
- Parses command-line arguments: bitrate, framerate, port, output size, crop area, input toggle
- Finds target window by name (supports "desktop" for entire screen, "cursor" for window under cursor)
- Creates and runs the application instance

#### 2. **Application Core** (`app.c/h`)
- Coordinates grabber, encoder, and server components
- Main rendering loop: grabs frame → encodes → broadcasts to all WebSocket clients
- Handles WebSocket events:
  - **HTTP requests**: Serves client files (index.html, jsmpg.js, jsmpg-vnc.js)
  - **Connection**: Sends initial header with video dimensions
  - **Messages**: Processes keyboard/mouse input from clients
  - **Disconnection**: Logs client disconnect

**Custom Protocol**:
- Each video frame has a custom header (not standard MPEG):
  ```c
  typedef struct {
      jsmpeg_trame_type_t type;  // 0xFA010000 for video
      int size;                   // Frame size
      char data[0];               // Encoded MPEG data
  } jsmpeg_frame_t;
  ```
- Initial handshake sends jsmpeg header: `{'j','s','m','p', width, height}`

#### 3. **Screen Grabber** (`grabber.c/h`)
- Uses Windows GDI (`BitBlt`) to capture window or desktop pixels
- Supports optional crop area to capture only part of a window
- Returns raw RGB pixels for encoding

#### 4. **Encoder** (`encoder.c/h`)
- Uses ffmpeg's libavcodec (MPEG1VIDEO codec)
- Scales video if output size differs from input
- Uses libswscale for RGB → YUV420P conversion
- Produces MPEG1 video packets

#### 5. **WebSocket Server** (`server.c/h`)
- Wraps libwebsockets for WebSocket communication
- Manages connected clients (linked list)
- Handles HTTP file serving and WebSocket protocol upgrades
- Broadcasts encoded frames to all connected clients

#### 6. **Browser Client** (`client/`)
- **jsmpg.js**: Modified jsmpeg decoder (custom frame header support)
- **jsmpg-vnc.js**: Input capture and WebSocket communication
  - Keyboard events → binary messages (input_key_t struct)
  - Mouse events → binary messages (input_mouse_t struct)
  - Supports absolute and relative mouse modes
  - Touch events for mobile devices

---

## Development Workflow

### Build System
- **Platform**: Windows only (uses Win32 API extensively)
- **IDE**: Visual Studio (solution file: `vs/jsmpeg-vnc.sln`)
- **Dependencies**:
  - ffmpeg libraries (precompiled, included in `source/ffmpeg/`)
  - libwebsockets (included in `source/libwebsocket/`)

### Building the Project
1. Open `vs/jsmpeg-vnc.sln` in Visual Studio
2. Build configuration: Release/Debug (x86 or x64)
3. Output: `jsmpeg-vnc.exe` in build directory

### Running the Application
```bash
jsmpeg-vnc.exe [options] <window name>

Options:
  -b bitrate in kilobit/s (default: estimated by output size)
  -s output size as WxH (default: same as window size)
  -f target framerate (default: 60)
  -p port (default: 8080)
  -c crop area as X,Y,W,H
  -i enable/disable remote input (default: 1)

Examples:
  jsmpeg-vnc.exe -b 2000 -s 640x480 -f 30 -p 9006 "Quake 3: Arena"
  jsmpeg-vnc.exe desktop
  jsmpeg-vnc.exe cursor
```

### Client Access
- Navigate to `http://<server-ip>:8080/` in a browser
- For mouse lock (relative mouse movement): `http://<server-ip>:8080/?mouselock`

---

## Code Conventions

### Naming Conventions
- **Files**: lowercase with underscores (e.g., `screen_grabber.c`)
- **Types**: `module_name_t` suffix (e.g., `encoder_t`, `grabber_t`)
- **Functions**: `module_function` pattern (e.g., `encoder_create`, `grabber_grab`)
- **Constants**: UPPERCASE_WITH_UNDERSCORES (e.g., `APP_MOUSE_SPEED`)
- **Enums**: lowercase_with_underscores (e.g., `input_type_key`)

### Struct Pattern
Most modules follow this pattern:
```c
// Header file (module.h)
typedef struct {
    // module state
} module_t;

module_t *module_create(...);
void module_destroy(module_t *self);
void module_operation(module_t *self, ...);

// Implementation file (module.c)
module_t *module_create(...) {
    module_t *self = (module_t *)malloc(sizeof(module_t));
    memset(self, 0, sizeof(module_t));
    // initialization
    return self;
}

void module_destroy(module_t *self) {
    if (self == NULL) { return; }
    // cleanup
    free(self);
}
```

### Memory Management
- All modules have `create` and `destroy` functions
- Manual memory management (malloc/free)
- Always check for NULL before accessing pointers
- Use `memset` to zero-initialize structs

### Error Handling
- Return NULL from `create` functions on failure
- Print error messages to stdout with `printf`
- Early returns for invalid input

### Windows API Usage
- Heavy use of Win32 API for window management, input injection, and GDI
- Extended key codes for special keys (arrows, numpad, etc.)
- `ClientToScreen` for coordinate transformation

---

## Key Files and Their Responsibilities

| File | Lines | Purpose |
|------|-------|---------|
| `source/jsmpeg-vnc.c` | ~130 | Entry point, CLI parsing, window finding |
| `source/app.c` | ~246 | Main application loop, event handling |
| `source/encoder.c` | ~80 | ffmpeg video encoding |
| `source/grabber.c` | ~90 | Windows screen capture |
| `source/server.c` | ~200 | WebSocket server management |
| `source/timer.c` | ~50 | High-precision timing |
| `client/index.html` | ~66 | Browser UI |
| `client/jsmpg-vnc.js` | ~132 | Input handling and WebSocket client |
| `client/jsmpg.js` | ~2500 | MPEG1 video decoder (third-party, modified) |

---

## Important Technical Details

### Video Encoding
- **Codec**: MPEG1VIDEO (old but low-latency, simple to decode in browser)
- **Pixel Format**: YUV420P
- **Bitrate**: Auto-estimated as `output_width * 1500 bps` if not specified
- **Frame Rate**: Target 60 fps (actual varies with CPU load)
- **Scaling**: Uses libswscale's SWS_FAST_BILINEAR algorithm

### Input Handling
- **Keyboard**: Uses scan codes via `MapVirtualKey` and `keybd_event`
- **Mouse**:
  - Absolute mode: Maps canvas coordinates to screen coordinates
  - Relative mode: Uses `mouse_event` with movement deltas
  - Mouse speed multiplier: `APP_MOUSE_SPEED` (5.0f)
- **Touch**: Simulates mouse events on mobile devices

### Network Protocol
All binary data uses **network byte order (big-endian)** for multi-byte values.

**Initial Handshake**:
```c
jsmpeg_header_t {
    magic: [0x6a, 0x73, 0x6d, 0x70]  // "jsmp"
    width: uint16 (big-endian)
    height: uint16 (big-endian)
}
```

**Video Frames**:
```c
jsmpeg_frame_t {
    type: uint32 = 0xFA010000 (big-endian)
    size: uint32 (big-endian) = sizeof(header) + data_length
    data: byte[]  // MPEG1 encoded data
}
```

**Input Messages (Client → Server)**:
```c
// Keyboard
input_key_t {
    type: uint16 = 0x0001
    state: uint16  // 0x00 = up, 0x01 = down
    key_code: uint16  // Windows VK code
}

// Mouse
input_mouse_t {
    type: uint16  // Bitfield: 0x0002=button, 0x0004=absolute, 0x0008=relative
    flags: uint16  // Mouse button flags
    x: float32
    y: float32
}
```

### Performance Considerations
- **Windows Aero**: Significantly slows down desktop capture; should be disabled
- **CPU Usage**: Encoding 1080p @ 60fps uses ~1 full CPU core
- **Latency**: Custom protocol removes 1 frame of latency vs. standard MPEG streams
- **Network**: Bitrate should be adjusted based on WiFi quality

---

## Testing Strategy

### Manual Testing
1. **Basic Functionality**
   - Capture desktop/window
   - Stream to browser
   - Verify video quality and framerate
   - Test keyboard/mouse input

2. **Edge Cases**
   - Window minimized or hidden
   - Window resized during streaming
   - Multiple concurrent clients
   - Network disconnection/reconnection

3. **Input Testing**
   - All keyboard keys (especially extended keys)
   - Mouse absolute vs. relative modes
   - Mouse lock for games
   - Touch controls on mobile

4. **Performance Testing**
   - Various resolutions (480p, 720p, 1080p)
   - Different framerates (30, 60 fps)
   - Multiple bitrates
   - CPU usage monitoring

### Common Issues
- **Missing MSVCR100.dll**: Install [Microsoft Visual C++ 2010 Redistributable](https://www.microsoft.com/en-us/download/details.aspx?id=5555)
- **Port already in use**: Change port with `-p` option
- **Black screen**: Window may be minimized or Aero interfering
- **High CPU**: Reduce resolution, framerate, or bitrate

---

## Extending the Project

### Adding New Command-Line Options
1. Add option parsing in `jsmpeg-vnc.c` main function (switch statement)
2. Pass value to `app_create` function
3. Store in `app_t` struct if needed across frames
4. Use in relevant module (grabber, encoder, server)

### Adding New Input Types
1. Define input type in `app.c` (e.g., `input_type_gamepad`)
2. Define struct layout matching client
3. Handle in `app_on_message` function
4. Update client (`jsmpg-vnc.js`) to send new input type

### Supporting New Platforms
- **Linux**: Replace GDI grabber with X11/Wayland, replace input injection
- **macOS**: Use Core Graphics for capture, CGEventPost for input
- **All platforms**: Consider using cross-platform libraries (SDL2, etc.)

### Improving Video Quality
- Switch to H.264 codec (requires browser support and more CPU)
- Adjust encoder quality settings (`bit_rate`, `qmin`, `qmax`)
- Use different pixel formats
- Implement adaptive bitrate based on network conditions

---

## Git Workflow

### Branching Strategy
- **Main branch**: `master` (or default branch)
- **Feature branches**: Use descriptive names (e.g., `feature/h264-encoding`, `fix/mouse-scaling`)
- **AI branches**: Prefix with `claude/` (e.g., `claude/add-claude-documentation-PLwD3`)

### Commit Guidelines
- Use clear, descriptive commit messages
- Focus on "why" rather than "what" (code shows what changed)
- Reference issue numbers if applicable
- Keep commits focused on single logical changes

### Pull Requests
- Ensure code compiles on Windows/Visual Studio
- Test all affected functionality
- Update README.md if user-facing changes
- Update CLAUDE.md if architectural changes

---

## AI Assistant Guidelines

### Before Making Changes
1. **Read existing code** - Never propose changes without reading the file first
2. **Understand the architecture** - Know how components interact
3. **Check dependencies** - Some changes affect multiple modules
4. **Consider Windows-specific APIs** - This is a Windows-only application

### Code Style
- Follow existing naming conventions
- Use `module_operation` naming pattern
- Add error checking for all allocations and API calls
- Clean up resources in reverse order of creation

### Common Tasks

#### Adding a Feature
1. Identify affected modules (app, encoder, grabber, server, client)
2. Design data structures and function signatures
3. Update headers first, then implementation
4. Update client if protocol changes
5. Test thoroughly

#### Fixing a Bug
1. Reproduce the issue
2. Locate the root cause (use printf debugging if needed)
3. Fix with minimal changes
4. Verify fix doesn't break other functionality

#### Refactoring
- Keep changes minimal and focused
- Ensure backward compatibility (especially protocol)
- Update all call sites if function signatures change
- Test after each logical refactoring step

### Security Considerations
- **Input Validation**: All user input (CLI args, WebSocket messages) should be validated
- **Buffer Overflows**: Check bounds on all string operations
- **Remote Input**: Be aware this allows remote control of the machine
- **Network Security**: No encryption (consider adding WSS support)

### Avoid
- Breaking changes to the binary protocol (client/server incompatibility)
- Platform-specific code that won't compile on Windows/MSVC
- Adding large dependencies without discussion
- Over-engineering simple features

---

## Dependencies and Licenses

### Third-Party Libraries
- **ffmpeg**: LGPL 2.1+ (https://github.com/FFmpeg/FFmpeg)
- **libwebsockets**: LGPL 2.1 with static linking exception (https://github.com/warmcat/libwebsockets)
- **jsmpeg**: MIT License (https://github.com/phoboslab/jsmpeg) - *Modified version*

### Project License
**GPL v3** (http://www.gnu.org/licenses/gpl-3.0.en.html)

### Note on jsmpeg Modification
The included jsmpeg decoder has been modified to handle custom frame headers, making WebSocket streams incompatible with standard MPEG video files. This modification reduces latency by one frame.

---

## Resources

- **Main Repository**: https://github.com/phoboslab/jsmpeg-vnc
- **Demo Video**: http://phoboslab.org/log/2015/07/play-gta-v-in-your-browser-sort-of
- **Binary Releases**: https://github.com/phoboslab/jsmpeg-vnc/releases
- **Author Contact**: dominic@phoboslab.org (for licensing or consulting)

---

## Troubleshooting

### Build Issues
- Ensure Visual Studio with C++ tools is installed
- Check that ffmpeg and libwebsocket libraries are in `source/` directories
- Verify project targets correct architecture (x86/x64)

### Runtime Issues
- Check Windows firewall isn't blocking the port
- Verify window name matches exactly (case-sensitive prefix match)
- For desktop capture, try disabling Windows Aero theme
- Monitor console output for error messages

### Client Issues
- Use Chrome/Firefox (modern browsers with WebSocket support)
- Check browser console for JavaScript errors
- Verify WebSocket connection (should see "ws://" in Network tab)
- Try disabling browser extensions

---

**Last Updated**: 2026-01-20
**Document Version**: 1.0
