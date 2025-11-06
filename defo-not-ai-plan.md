# Capture Card Media Player - Development Roadmap

## Phase 1: Project Foundation

### Initial Setup

- Create new Rust project with cargo
- Set up project structure with platform-specific modules (windows, linux, common)
- Configure Cargo.toml with core dependencies: wgpu, winit, egui, nokhwa/v4l, cpal
- Set up feature flags for platform-specific code (windows, linux, asio)
- Configure build profiles for development and release
- Initialize version control and .gitignore

### Basic Window and Rendering Pipeline

- Create main window using winit
- Initialize wgpu device and surface with proper present mode configuration
- Set up basic render loop with frame timing
- Implement simple texture rendering to verify GPU pipeline works
- Add keyboard input handling for exit and basic controls
- Verify window runs on both target platforms

### Milestone: Working empty window with render loop

## Phase 2: Video Capture Core

### Device Enumeration

- Implement video device discovery using nokhwa or platform APIs
- Create device information structures (name, capabilities, formats)
- Build device selection interface (temporary console output is fine)
- Add hotplug detection for device connection/disconnection
- Verify capture card detection on both platforms

### Basic Video Capture

- Initialize capture device with default settings
- Set up frame callback or polling mechanism
- Allocate frame buffers for incoming video data
- Implement basic frame acquisition loop
- Add error handling for device failures
- Test with actual capture card connected

### Video Display Integration

- Upload captured frames to GPU textures
- Implement YUV to RGB color conversion (compute shader)
- Render video frames to window surface
- Handle aspect ratio and scaling
- Add frame drop detection and logging
- Verify smooth video playback

### Milestone: Live video from capture card displaying in window

## Phase 3: Audio Capture and Playback

### Audio Device Setup

- Enumerate audio input devices using CPAL
- Match audio device to video source (usually same device)
- Configure audio stream with fixed buffer size (256 samples)
- Enable ASIO support on Windows for low latency
- Test audio device detection and initialization

### Audio Capture Pipeline

- Set up audio input stream with callback
- Implement lock-free ring buffer for audio data
- Create audio processing thread for non-blocking operation
- Add buffer overflow/underflow detection
- Test audio capture independently from video

### Audio Playback

- Configure audio output stream
- Connect ring buffer to output callback
- Implement basic volume control
- Add audio routing (capture device to speakers/headphones)
- Verify audio plays without glitches or dropouts

### Milestone: Audio and video both working (not yet synchronized)

## Phase 4: Audio-Video Synchronization

### Timestamp Management

- Implement timestamp tracking for video frames
- Add timestamp tracking for audio samples
- Create synchronization state structure
- Build drift detection mechanism
- Add logging for A/V sync metrics

### Synchronization Implementation

- Designate audio as timing master
- Calculate offset between audio and video timelines
- Implement frame dropping for video running ahead
- Implement frame duplication for video running behind
- Add drift compensation with threshold-based adjustment
- Test with various content types (gaming, video playback)

### Milestone: Synchronized audio and video playback

## Phase 5: User Interface

### Settings Menu Foundation

- Integrate egui with existing wgpu rendering
- Create overlay toggle (keyboard shortcut)
- Build main settings window structure
- Implement settings panel layout

### Device Selection UI

- Add video device dropdown with live enumeration
- Add audio device dropdown
- Implement device switching without restart
- Show device status indicators (connected/disconnected)
- Add format and resolution selection

### Playback Controls

- Implement volume slider with visual feedback
- Add volume mute toggle
- Create aspect ratio controls (fit, fill, stretch, native)
- Add latency/buffer size adjustment controls
- Implement fullscreen toggle button

### Visual Polish

- Add framerate display (optional debug overlay)
- Implement device information panel
- Add connection status indicators
- Create error message display system
- Design minimal, non-intrusive UI appearance

### Milestone: Fully functional settings interface

## Phase 6: Configuration Persistence

### Settings System

- Define configuration structure with all user preferences
- Implement settings loading with confy
- Implement settings saving on change
- Add default values for first launch
- Handle missing or corrupted config files gracefully

### State Management

- Save last selected video device
- Save last selected audio device
- Persist volume level
- Store window position and size
- Remember fullscreen state
- Save aspect ratio preference
- Store any custom latency settings

### Device Persistence Strategy

- Store device identifiers (not indices)
- Implement fallback to default device if saved device unavailable
- Add validation for loaded settings
- Create settings migration system for future updates

### Milestone: Settings persist across application restarts

## Phase 7: Window Management

### Fullscreen Implementation

- Implement borderless fullscreen mode
- Add keyboard shortcut for fullscreen toggle (F11)
- Handle monitor selection for multi-monitor setups
- Implement smooth transitions between windowed and fullscreen
- Preserve window position when returning from fullscreen

### Window State Management

- Handle window resize events properly
- Maintain aspect ratio during resize (if enabled)
- Implement window minimization/restoration
- Add window focus handling
- Handle DPI scaling on high-DPI displays

### Milestone: Polished window management experience

## Phase 8: Performance Optimization

### Latency Optimization

- Profile application with cargo-flamegraph
- Identify bottlenecks in capture-to-display pipeline
- Optimize buffer sizes for minimal latency
- Fine-tune GPU present mode and frame latency
- Minimize memory allocations in hot paths
- Implement zero-copy paths where possible

### Threading and Concurrency

- Verify proper thread separation (capture, processing, display)
- Implement lock-free communication between threads
- Add thread priority settings where platform supports
- Optimize channel buffer sizes for backpressure
- Profile thread utilization and balance workload

### Memory Management

- Pre-allocate all frame buffers at initialization
- Implement buffer pooling for reusable resources
- Eliminate runtime allocations from hot paths
- Profile memory usage and optimize footprint
- Add memory leak detection in debug builds

### Milestone: Consistent sub-100ms latency achieved

## Phase 9: Error Handling and Robustness

### Device Error Handling

- Handle device disconnection gracefully
- Implement automatic reconnection attempts
- Add user notifications for device errors
- Create fallback behavior for missing devices
- Log errors for troubleshooting

### Format and Compatibility

- Handle unsupported video formats gracefully
- Add format conversion where necessary
- Test with various capture card models
- Handle unusual resolutions and frame rates
- Add warnings for suboptimal configurations

### Application Stability

- Add panic handlers with graceful shutdown
- Implement proper resource cleanup on exit
- Handle GPU device loss scenarios
- Add recovery mechanisms for transient failures
- Create comprehensive error logging

### Milestone: Stable application that handles edge cases

## Phase 10: Cross-Platform Testing

### Windows Testing

- Test with DirectShow and Media Foundation backends
- Verify ASIO audio support works
- Test with various USB and PCIe capture cards
- Validate on different Windows versions (10, 11)
- Check high-DPI display support

### Linux Testing

- Test with V4L2 on various distributions
- Verify ALSA and PulseAudio/PipeWire compatibility
- Test with different kernel versions
- Validate with various capture card drivers
- Check Wayland and X11 compositor compatibility

### Hardware Compatibility

- Test with USB 2.0 and USB 3.0 capture cards
- Validate with different resolution/framerate combinations
- Test with UVC-compliant devices
- Verify with professional capture cards
- Document known compatibility issues

### Milestone: Working reliably on both target platforms

## Phase 11: User Experience Polish

### Visual Refinements

- Add smooth fade transitions for UI elements
- Implement loading states during device initialization
- Add visual feedback for all user actions
- Create help tooltips for settings
- Design informative error messages

### Performance Indicators

- Add optional latency display
- Show dropped frame counter
- Display current resolution and framerate
- Add audio buffer health indicators
- Create debug overlay for troubleshooting

### Keyboard Shortcuts

- Implement fullscreen toggle (F11)
- Add settings menu toggle (Escape or custom key)
- Create volume adjustment shortcuts
- Add aspect ratio cycling
- Implement mute toggle

### Milestone: Polished, user-friendly application

## Phase 12: Documentation and Packaging

### Documentation

- Write README with feature overview
- Create installation instructions for both platforms
- Document system requirements
- Add troubleshooting guide for common issues
- List tested capture card models
- Include build instructions from source

### Build System

- Configure release builds with optimizations
- Set up cross-compilation if needed
- Create platform-specific build scripts
- Optimize binary size
- Strip debug symbols for release

### Distribution

- Create Windows installer or portable executable
- Build Linux binaries or AppImage
- Write release notes
- Consider package manager distribution (optional)
- Set up update mechanism (future consideration)

### Milestone: Distributable application ready for users

## Phase 13: Future Enhancements (Optional)

### Advanced Features

- Recording capability to disk
- Screenshot functionality
- Multiple capture card support
- Picture-in-picture mode
- Stream output (RTMP/WebRTC)
- Hardware encoding integration
- Custom shader effects
- Overlay graphics support

### Performance Features

- HDR video support
- 4K resolution optimization
- High framerate support (120fps+)
- Hardware-accelerated processing
- GPU-specific optimizations

### User Requests

- Monitor community feedback
- Prioritize most requested features
- Address reported bugs
- Improve compatibility based on user testing

---

## General Development Principles

**Iterate in small steps**: Get each phase fully working before moving to the next

**Test frequently**: Verify on both platforms regularly, don't wait until the end

**Profile early**: Measure latency from the beginning, don't optimize blindly

**Handle errors**: Add error handling as you build features, not as an afterthought

**Keep it simple**: Start with the minimal viable feature, add complexity only when needed

**Document decisions**: Note why you chose specific approaches for future reference

**Use version control**: Commit working states frequently, especially before major changes

This roadmap takes you from empty project to polished, cross-platform capture card viewer with low latency and professional quality. Each milestone represents a functional checkpoint where you can test, evaluate, and adjust before proceeding.
