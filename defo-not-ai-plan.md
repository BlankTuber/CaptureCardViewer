# Capture Card Viewer - Concrete Development Plan

**Date:** October 28, 2025

## Executive Summary

This is a step-by-step development plan with **specific libraries** and **concrete implementation steps** for your Capture Card Viewer. Based on current Rust ecosystem research, I've identified the exact tools you need and broken down the project into manageable chunks you can tackle one at a time.

**Language Decision: Use Rust.** It has everything you need - modern tooling, great package manager (Cargo), and excellent learning resources.

---

## Technology Stack (Decided)

### Core Libraries

**Video Capture:**

- **`nokhwa` (v0.10+)** - Cross-platform camera/capture card library
  - Works on Windows (DirectShow/Media Foundation) and Linux (V4L2)
  - Supports most USB capture cards
  - Easy device enumeration
  - GitHub: <https://github.com/l1npengtul/nokhwa>
  - Docs: <https://docs.rs/nokhwa>

**Audio Playback:**

- **`cpal` (latest)** - Cross-platform audio I/O
  - Low-level, gives you control over latency
  - Works on Windows (WASAPI) and Linux (ALSA/PulseAudio)
  - Industry standard for Rust audio
  - GitHub: <https://github.com/RustAudio/cpal>
  - Docs: <https://docs.rs/cpal>

**Video Rendering:**

- **`sdl2` (v0.38+)** - Simple DirectMedia Layer 2 bindings
  - Fast texture-based rendering
  - Cross-platform window management
  - Handles video efficiently
  - GitHub: <https://github.com/Rust-SDL2/rust-sdl2>
  - Docs: <https://rust-sdl2.github.io/rust-sdl2/sdl2/>

**GUI Framework:**

- **`egui` (latest) with `eframe`** - Immediate mode GUI
  - Super simple to use (perfect for learning)
  - Great for simple UIs like yours
  - Works with SDL2
  - GitHub: <https://github.com/emilk/egui>
  - Docs: <https://docs.rs/egui>

**Configuration:**

- **`serde` + `toml`** - For saving settings
- **`anyhow`** - For easy error handling

---

## Project Structure

```md
capture-card-viewer/
├── Cargo.toml
├── src/
│   ├── main.rs           # Entry point
│   ├── capture.rs        # Video/audio capture logic
│   ├── display.rs        # Video rendering with SDL2
│   ├── audio.rs          # Audio playback with cpal
│   ├── gui.rs            # GUI with egui
│   ├── config.rs         # Settings management
│   └── error.rs          # Custom error types
└── config.toml           # User settings file
```

---

## Phase-by-Phase Implementation

### Phase 0: Setup & Hello World (1-2 days)

**Goal:** Get Rust and basic project running

1. **Create Project:**

   ```bash
   cargo new capture-card-viewer
   cd capture-card-viewer
   ```

2. **Install SDL2 Native Libraries:**

   **Windows:**

   ```bash
   # SDL2 will be bundled with the rust crate when using "bundled" feature
   # No separate installation needed
   ```

   **Linux:**

   ```bash
   # Ubuntu/Debian
   sudo apt-get install libsdl2-dev libsdl2-image-dev libsdl2-ttf-dev
   
   # Fedora
   sudo dnf install SDL2-devel SDL2_image-devel SDL2_ttf-devel
   
   # For audio (ALSA)
   sudo apt-get install libasound2-dev  # Ubuntu/Debian
   sudo dnf install alsa-lib-devel      # Fedora
   ```

3. **Test Basic Rust:**

   ```rust
   // src/main.rs
   fn main() {
       println!("Capture Card Viewer starting...");
   }
   ```

   Run with: `cargo run`

4. **Test SDL2 Window:**

   ```rust
   // src/main.rs
   use sdl2::event::Event;
   use sdl2::keyboard::Keycode;c
   use std::time::Duration;

   fn main() -> Result<(), String> {
       let sdl_context = sdl2::init()?;
       let video_subsystem = sdl_context.video()?;
       
       let window = video_subsystem
           .window("Test Window", 800, 600)
           .position_centered()
           .build()
           .map_err(|e| e.to_string())?;
       
       let mut canvas = window.into_canvas().build().map_err(|e| e.to_string())?;
       let mut event_pump = sdl_context.event_pump()?;
       
       'running: loop {
           for event in event_pump.poll_iter() {
               match event {
                   Event::Quit {..} | Event::KeyDown { keycode: Some(Keycode::Escape), .. } => {
                       break 'running
                   },
                   _ => {}
               }
           }
           
           canvas.clear();
           canvas.present();
           ::std::thread::sleep(Duration::new(0, 1_000_000_000u32 / 60));
       }
       
       Ok(())
   }
   ```

**Milestone:** You have a blank window that opens and closes. This confirms SDL2 works.

---

### Phase 1: Device Enumeration (2-3 days)

**Goal:** List all available capture devices

1. **Add nokhwa to Cargo.toml** (already in the setup above)

2. **Create capture.rs:**

   ```rust
   // src/capture.rs
   use nokhwa::pixel_format::RgbFormat;
   use nokhwa::utils::{CameraIndex, RequestedFormat, RequestedFormatType};
   use nokhwa::Camera;
   use anyhow::Result;

   pub fn list_devices() -> Result<Vec<String>> {
       let devices = nokhwa::query_devices(nokhwa::utils::ApiBackend::Auto)?;
       let device_names: Vec<String> = devices
           .iter()
           .map(|info| info.human_name().to_string())
           .collect();
       Ok(device_names)
   }

   pub struct CaptureDevice {
       camera: Camera,
   }

   impl CaptureDevice {
       pub fn new(device_index: usize) -> Result<Self> {
           let index = CameraIndex::Index(device_index as u32);
           
           // Request 1080p@60fps, RGB format
           let requested = RequestedFormat::new::<RgbFormat>(
               RequestedFormatType::AbsoluteHighestFrameRate
           );
           
           let camera = Camera::new(index, requested)?;
           
           Ok(Self { camera })
       }
       
       pub fn start(&mut self) -> Result<()> {
           self.camera.open_stream()?;
           Ok(())
       }
       
       pub fn get_frame(&mut self) -> Result<Vec<u8>> {
           let frame = self.camera.frame()?;
           let decoded = frame.decode_image::<RgbFormat>()?;
           Ok(decoded.to_vec())
       }
   }
   ```

3. **Update main.rs to list devices:**

   ```rust
   // src/main.rs
   mod capture;
   use anyhow::Result;

   fn main() -> Result<()> {
       println!("Searching for capture devices...");
       
       let devices = capture::list_devices()?;
       
       if devices.is_empty() {
           println!("No capture devices found!");
           return Ok(());
       }
       
       println!("Found {} device(s):", devices.len());
       for (i, device) in devices.iter().enumerate() {
           println!("  [{}] {}", i, device);
       }
       
       Ok(())
   }
   ```

4. **Test:**

   ```bash
   cargo run
   ```

   **Expected output:** List of your capture cards/webcams

**Milestone:** Your program can detect your capture card.

---

### Phase 2: Video Capture & Display (4-5 days)

**Goal:** Capture frames and display them in a window

1. **Create display.rs:**

   ```rust
   // src/display.rs
   use sdl2::render::{Canvas, Texture, TextureCreator};
   use sdl2::video::{Window, WindowContext};
   use sdl2::pixels::PixelFormatEnum;
   use anyhow::Result;

   pub struct VideoDisplay {
       canvas: Canvas<Window>,
       texture_creator: TextureCreator<WindowContext>,
   }

   impl VideoDisplay {
       pub fn new(width: u32, height: u32) -> Result<Self> {
           let sdl_context = sdl2::init()
               .map_err(|e| anyhow::anyhow!("SDL init failed: {}", e))?;
           let video_subsystem = sdl_context.video()
               .map_err(|e| anyhow::anyhow!("Video subsystem failed: {}", e))?;
           
           let window = video_subsystem
               .window("Capture Card Viewer", width, height)
               .position_centered()
               .resizable()
               .build()?;
           
           let canvas = window.into_canvas().build()
               .map_err(|e| anyhow::anyhow!("Canvas creation failed: {}", e))?;
           
           let texture_creator = canvas.texture_creator();
           
           Ok(Self {
               canvas,
               texture_creator,
           })
       }
       
       pub fn render_frame(&mut self, frame_data: &[u8], width: u32, height: u32) -> Result<()> {
           // Create texture for this frame
           let mut texture = self.texture_creator
               .create_texture_streaming(PixelFormatEnum::RGB24, width, height)
               .map_err(|e| anyhow::anyhow!("Texture creation failed: {}", e))?;
           
           // Update texture with frame data
           texture.update(None, frame_data, (width * 3) as usize)
               .map_err(|e| anyhow::anyhow!("Texture update failed: {}", e))?;
           
           // Clear and render
           self.canvas.clear();
           self.canvas.copy(&texture, None, None)
               .map_err(|e| anyhow::anyhow!("Canvas copy failed: {}", e))?;
           self.canvas.present();
           
           Ok(())
       }
   }
   ```

2. **Update main.rs to capture and display:**

   ```rust
   // src/main.rs
   mod capture;
   mod display;
   
   use anyhow::Result;
   use sdl2::event::Event;
   use sdl2::keyboard::Keycode;
   use std::time::Duration;

   fn main() -> Result<()> {
       // List and select device
       let devices = capture::list_devices()?;
       if devices.is_empty() {
           println!("No devices found");
           return Ok(());
       }
       
       println!("Using device: {}", devices[0]);
       
       // Create capture device (device index 0)
       let mut capture_device = capture::CaptureDevice::new(0)?;
       capture_device.start()?;
       
       // Create display window (1920x1080 for now)
       let mut display = display::VideoDisplay::new(1920, 1080)?;
       
       // Get SDL event pump for handling window events
       let sdl_context = sdl2::init().unwrap();
       let mut event_pump = sdl_context.event_pump().unwrap();
       
       println!("Starting capture loop...");
       
       'running: loop {
           // Handle events
           for event in event_pump.poll_iter() {
               match event {
                   Event::Quit {..} | 
                   Event::KeyDown { keycode: Some(Keycode::Escape), .. } => {
                       break 'running
                   },
                   _ => {}
               }
           }
           
           // Capture and display frame
           match capture_device.get_frame() {
               Ok(frame_data) => {
                   // Assume 1920x1080 for now - we'll get actual dimensions later
                   if let Err(e) = display.render_frame(&frame_data, 1920, 1080) {
                       eprintln!("Display error: {}", e);
                   }
               }
               Err(e) => {
                   eprintln!("Capture error: {}", e);
               }
           }
           
           // Small delay to prevent 100% CPU usage
           ::std::thread::sleep(Duration::from_millis(1));
       }
       
       println!("Shutting down...");
       Ok(())
   }
   ```

3. **Test with your capture card:**

   ```bash
   cargo run
   ```

**Milestone:** You see video from your capture card in the window!

---

### Phase 3: Audio Capture & Playback (3-4 days)

**Goal:** Get audio working and in sync with video

1. **Create audio.rs:**

   ```rust
   // src/audio.rs
   use cpal::traits::{DeviceTrait, HostTrait, StreamTrait};
   use cpal::{Device, Stream, StreamConfig};
   use anyhow::Result;
   use std::sync::{Arc, Mutex};

   pub struct AudioPlayer {
       _stream: Stream,
       device: Device,
   }

   impl AudioPlayer {
       pub fn new() -> Result<Self> {
           let host = cpal::default_host();
           let device = host.default_output_device()
               .ok_or_else(|| anyhow::anyhow!("No output device available"))?;
           
           let config = device.default_output_config()?;
           
           println!("Audio output device: {}", device.name()?);
           println!("Audio config: {:?}", config);
           
           // Create a simple sine wave for testing
           let mut sample_clock = 0f32;
           let sample_rate = config.sample_rate().0 as f32;
           
           let stream = device.build_output_stream(
               &config.into(),
               move |data: &mut [f32], _: &cpal::OutputCallbackInfo| {
                   for sample in data.iter_mut() {
                       sample_clock = (sample_clock + 1.0) % sample_rate;
                       *sample = (sample_clock * 440.0 * 2.0 * std::f32::consts::PI / sample_rate).sin();
                   }
               },
               move |err| {
                   eprintln!("Audio stream error: {}", err);
               },
               None
           )?;
           
           stream.play()?;
           
           Ok(Self {
               _stream: stream,
               device,
           })
       }
   }
   
   // Later, you'll integrate actual audio from the capture card
   // For now, this is a placeholder that proves audio works
   ```

2. **Update main.rs to include audio:**

   ```rust
   // Add to main.rs
   mod audio;
   
   // In main() function, add:
   let _audio = audio::AudioPlayer::new()?;
   println!("Audio started (test tone)");
   ```

**Note:** At this stage, audio is separate from video. You'll hear a test tone while seeing video. Syncing comes next.

**Milestone:** You hear audio output (test tone) while video plays.

---

### Phase 4: Audio-Video Sync (3-4 days)

**Goal:** Make audio and video play in sync

This is one of the trickier parts. Here's the approach:

1. **Add timestamp tracking to capture.rs:**

   ```rust
   use std::time::Instant;
   
   pub struct Frame {
       pub data: Vec<u8>,
       pub timestamp: Instant,
       pub width: u32,
       pub height: u32,
   }
   
   impl CaptureDevice {
       pub fn get_frame_with_timestamp(&mut self) -> Result<Frame> {
           let timestamp = Instant::now();
           let frame = self.camera.frame()?;
           let decoded = frame.decode_image::<RgbFormat>()?;
           
           Ok(Frame {
               data: decoded.to_vec(),
               timestamp,
               width: frame.width(),
               height: frame.height(),
           })
       }
   }
   ```

2. **Use a ring buffer for audio-video sync:**

   ```rust
   // This is simplified - real implementation needs careful buffer management
   use std::collections::VecDeque;
   use std::sync::{Arc, Mutex};
   
   pub struct SyncBuffer {
       video_frames: Arc<Mutex<VecDeque<Frame>>>,
       audio_samples: Arc<Mutex<VecDeque<f32>>>,
   }
   ```

3. **Calculate and correct drift:**
   - Track video presentation time
   - Track audio playback time
   - If video is ahead, drop frames
   - If audio is ahead, insert silence

**This is complex**, so for MVP, you might want to:

- Accept some drift (< 100ms is okay for many uses)
- Start simple: just display frames as fast as they come
- Add sync correction in v1.1

**Milestone:** Audio and video play together with acceptable sync (<100ms drift).

---

### Phase 5: Basic GUI (3-4 days)

**Goal:** Add device selection dropdown and basic controls

1. **Create gui.rs with egui:**

   ```rust
   // src/gui.rs
   use eframe::egui;
   
   pub struct ViewerApp {
       selected_device: usize,
       available_devices: Vec<String>,
       is_playing: bool,
       volume: f32,
   }
   
   impl Default for ViewerApp {
       fn default() -> Self {
           Self {
               selected_device: 0,
               available_devices: Vec::new(),
               is_playing: false,
               volume: 1.0,
           }
       }
   }
   
   impl eframe::App for ViewerApp {
       fn update(&mut self, ctx: &egui::Context, _frame: &mut eframe::Frame) {
           egui::TopBottomPanel::top("top_panel").show(ctx, |ui| {
               ui.horizontal(|ui| {
                   ui.label("Device:");
                   egui::ComboBox::from_id_salt("device_selector")
                       .selected_text(&self.available_devices[self.selected_device])
                       .show_ui(ui, |ui| {
                           for (i, device) in self.available_devices.iter().enumerate() {
                               ui.selectable_value(&mut self.selected_device, i, device);
                           }
                       });
                   
                   if ui.button(if self.is_playing { "Stop" } else { "Start" }).clicked() {
                       self.is_playing = !self.is_playing;
                   }
                   
                   ui.separator();
                   ui.label("Volume:");
                   ui.add(egui::Slider::new(&mut self.volume, 0.0..=1.0));
               });
           });
           
           egui::CentralPanel::default().show(ctx, |ui| {
               ui.heading("Video will display here");
               // Video rendering happens via SDL2 in a separate window for now
               // Later you can integrate egui with SDL2 or use egui's image support
           });
       }
   }
   ```

2. **For MVP, keep GUI simple:**
   - Have SDL2 window for video
   - Small egui window for controls
   - They can be separate windows initially

**Milestone:** You have basic UI controls that work.

---

### Phase 6: Linux Support (2-3 days)

**Goal:** Make it work on Linux

The great news: **nokhwa** and **cpal** handle most platform differences!

1. **Test on Linux:**

   ```bash
   # On Linux machine
   cargo build
   cargo run
   ```

2. **Handle Linux-specific issues:**
   - Check V4L2 permissions: `sudo usermod -a -G video $USER`
   - Check ALSA permissions: `sudo usermod -a -G audio $USER`
   - Test with different capture cards

3. **Add Linux-specific config if needed:**

   ```rust
   #[cfg(target_os = "linux")]
   fn setup_linux() {
       // Any Linux-specific initialization
   }
   ```

**Milestone:** Application runs on both Windows and Linux.

---

### Phase 7: Polish & Settings (2-3 days)

**Goal:** Add configuration, improve UX

1. **Create config.rs:**

   ```rust
   // src/config.rs
   use serde::{Deserialize, Serialize};
   use std::fs;
   use anyhow::Result;
   
   #[derive(Serialize, Deserialize, Default)]
   pub struct Config {
       pub default_device: Option<usize>,
       pub volume: f32,
       pub window_width: u32,
       pub window_height: u32,
       pub fullscreen_on_start: bool,
   }
   
   impl Config {
       pub fn load() -> Result<Self> {
           let config_str = fs::read_to_string("config.toml")?;
           let config: Config = toml::from_str(&config_str)?;
           Ok(config)
       }
       
       pub fn save(&self) -> Result<()> {
           let config_str = toml::to_string_pretty(self)?;
           fs::write("config.toml", config_str)?;
           Ok(())
       }
       
       pub fn load_or_default() -> Self {
           Self::load().unwrap_or_default()
       }
   }
   ```

2. **Add keyboard shortcuts:**
   - F for fullscreen
   - ESC to exit
   - M to mute
   - Space to pause (if implementing)

3. **Improve error messages:**
   - "No capture card detected. Please connect a capture device."
   - "Audio device not found. Check system audio settings."

**Milestone:** User-friendly application with saved preferences.

---

## Testing Strategy

### Per-Phase Testing

1. **Phase 1:** Plug/unplug capture cards, verify detection
2. **Phase 2:** Test with different resolutions, check frame drops
3. **Phase 3:** Verify audio doesn't crackle or skip
4. **Phase 4:** Watch for A/V drift over 5+ minutes
5. **Phase 5:** Click all UI elements, verify responsiveness
6. **Phase 6:** Test on actual Linux machine
7. **Phase 7:** Test with config file changes

### Integration Testing

- Run for 30+ minutes continuously
- Monitor CPU/memory usage
- Test device switching without restart
- Test with multiple capture card brands

---

## Common Issues & Solutions

### Issue: "No devices found"

**Solution:**

- Windows: Install capture card drivers
- Linux: Check `v4l2-ctl --list-devices`
- Verify capture card is plugged in and powered

### Issue: "Black screen"

**Solution:**

- Check if capture card needs input signal
- Verify resolution matches
- Try different device index

### Issue: "Audio crackling"

**Solution:**

- Increase buffer size in cpal
- Check CPU usage (should be <30%)
- Reduce video frame rate if needed

### Issue: "High CPU usage"

**Solution:**

- Use hardware decoding if available
- Reduce video resolution
- Optimize frame rendering (only update when new frame)

### Issue: "Build fails on Linux"

**Solution:**

```bash
# Install all required dev packages
sudo apt-get install build-essential libsdl2-dev libasound2-dev pkg-config
```

---

## Timeline Estimate

**MVP (Version 1.0):**

- Phase 0: 1-2 days
- Phase 1: 2-3 days
- Phase 2: 4-5 days
- Phase 3: 3-4 days
- Phase 4: 3-4 days (or defer to v1.1)
- Phase 5: 3-4 days
- Phase 6: 2-3 days
- Phase 7: 2-3 days

**Total: ~20-28 days** (3-4 weeks of focused work)

This assumes:

- Working ~4-6 hours per day
- Some debugging time
- Learning Rust as you go

---

## Learning Resources

### Rust Basics

- **The Rust Book:** <https://doc.rust-lang.org/book/>
- Focus on: Chapters 1-10, 13 (important!)
- **Rust by Example:** <https://doc.rust-lang.org/rust-by-example/>

### Library-Specific

- **nokhwa examples:** <https://github.com/l1npengtul/nokhwa/tree/master/examples>
- **cpal examples:** <https://github.com/RustAudio/cpal/tree/master/examples>
- **SDL2 Rust tutorial:** <https://blog.logrocket.com/using-sdl2-bindings-rust/>
- **egui demo:** <https://www.egui.rs/>

### When Stuck

- **Rust Discord:** <https://discord.gg/rust-lang>
- **r/rust subreddit:** <https://reddit.com/r/rust>
- **Rust Users Forum:** <https://users.rust-lang.org/>

---

## Next Steps - Start Here

1. **Today:** Install Rust, create project, get hello world running
2. **Tomorrow:** Get SDL2 window working
3. **Day 3:** Install capture card, test device enumeration
4. **Day 4:** Capture first frame and display it
5. **Day 5:** Get continuous video working

**Don't try to do everything at once.** Build one feature, get it working, then move to the next.

---

## Future Enhancements (Post-MVP)

After v1.0 is stable, consider:

### v1.1 - Improvements

- Better A/V sync with drift correction
- Hardware-accelerated video decoding
- Multi-threaded frame processing
- Performance metrics display

### v1.2 - Features

- Basic recording to MP4
- Screenshot capture
- Window always-on-top
- System tray integration

### v2.0 - Advanced

- Multiple capture cards support
- Streaming output (RTMP)
- Video filters (brightness, contrast)
- Audio device selection separate from video

---

## Critical Tips

1. **Start Simple:** Don't try to make everything perfect. Get it working first.

2. **Test Early:** Test each phase with your actual capture card before moving on.

3. **Use Examples:** The libraries have example code - copy and modify them.

4. **Ask for Help:** The Rust community is friendly. Use Discord/forums when stuck.

5. **Commit Often:** Use git, commit after each working feature.

6. **Document Weird Issues:** When you fix something tricky, write it down.

7. **Don't Optimize Early:** Get it working, then make it fast.

---

## Success Criteria (MVP)

You're done with v1.0 when:

- ✅ App launches and shows UI
- ✅ Capture card detected and listed
- ✅ Video displays with no visible lag
- ✅ Audio plays in sync with video
- ✅ Can switch devices from UI
- ✅ Runs stable for 30+ minutes
- ✅ Works on Windows (and ideally Linux)
- ✅ A friend can download and use it

---

## Final Words

This is a **learning project**, so expect:

- Some frustration with Rust's ownership system (totally normal!)
- Trial and error with A/V sync (everyone struggles with this)
- Platform-specific quirks (especially on Linux)
- Rewrites as you understand things better

But you'll learn:

- Real-time audio/video programming
- Rust's powerful type system
- Cross-platform development
- How capture cards actually work

---

## Appendix: Quick Command Reference

```bash
# Create project
cargo new capture-card-viewer

# Add a dependency
cargo add nokhwa --features input-native

# Build project
cargo build

# Run project
cargo run

# Build optimized release
cargo build --release

# Run release version
cargo run --release

# Check for errors without building
cargo check

# Format code
cargo fmt

# Run linter
cargo clippy
```
