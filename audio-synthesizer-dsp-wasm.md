## 1. Project Title

Audio Synthesizer and DSP with WebAssembly

## 2. Difficulty

Mid-Level

### Rationale
This project bridges low-level digital signal processing (DSP) and high-performance browser execution via WebAssembly (WASM). The developer must write memory-efficient audio processing loops in a systems language (like C, C++, or Rust), compile them to WASM, and interface with the browser's Web Audio API (`AudioWorkletNode`) to generate real-time synthesized waveforms without audio dropouts or buffer underruns.

## 3. Project Overview

The Audio Synthesizer and DSP WASM project is an interactive, browser-based modular synthesizer. It computes audio samples on-the-fly using custom oscillators (Sine, Square, Sawtooth, Triangle), envelope generators (ADSR), and digital filters (Low-pass, High-pass) compiled entirely to WebAssembly. By running inside a dedicated browser audio worklet thread, it guarantees low-latency sample generation, offering a tactile virtual keyboard and parameter knobs to manipulate sound in real time.

## 4. Problem Statement

Standard JavaScript runs on the main browser thread or generic worker threads, which are subject to garbage collection pauses and event loop congestion. 
- When building real-time audio applications in pure JavaScript, performance bottlenecks and GC pauses cause audible audio clicking, stuttering, and high latency.
- Implementing complex DSP math (like oscillators, biquad filters, and delay lines) in JavaScript is computationally expensive.
- Developers need a reliable way to write high-performance audio synthesis engines that run predictably in the browser.

## 5. Proposed Solution

The proposed software architecture combines WebAssembly and the Web Audio API:
1. **DSP Engine (WASM)**: Written in C++ or Rust, containing oscillator math, filter algorithms, and buffer generators.
2. **AudioWorklet**: A specialized background audio thread in the browser that fetches audio buffers from the WASM module at fixed sample rates (e.g., 44.1kHz).
3. **UI Frontend**: A web interface with piano keys, sliders, and knobs that dispatches parameter changes (`frequency`, `resonance`, `cutoff`) to the WASM engine via shared memory or message passing.

## 6. Project Goal

To build a zero-latency browser synthesizer powered by a custom WebAssembly DSP engine that generates real-time audio signals and responds smoothly to user input.

## 7. Core Workflow

```text
User Input (UI Sliders / Keys) ──> Main Thread ──(Message)──> AudioWorklet Thread
                                                                       │
                                                                       ▼
                                                          WASM DSP Audio Generator
                                                          (Computes raw PCM samples)
                                                                       │
                                                                       ▼
                                                           Web Audio API Output Buffer
                                                                       │
                                                                       ▼
                                                           Computer Audio Speakers
```

## 8. Functional Requirements

### DSP & Synthesis (WASM)
- **Oscillators**: Generate Sine, Square, Sawtooth, and Triangle waveforms at configurable frequencies.
- **ADSR Envelope**: Implement Attack, Decay, Sustain, and Release envelope modulation for amplitude shaping.
- **Biquad Filters**: Implement Low-pass and High-pass filters with adjustable cutoff frequency and resonance ($Q$).

### Audio Worklet Integration
- **Real-Time Buffer Filling**: Process audio in small blocks (e.g., 128 or 512 samples) to maintain ultra-low latency.
- **Thread Communication**: Handle parameter updates from the UI thread without locking the audio thread.

### User Interface
- **Virtual Keyboard**: On-screen or computer-keyboard music playing interface.
- **Parameter Controls**: Sliders and knobs to manipulate synthesizer settings in real time.

## 9. Non-Functional Requirements

### Performance & Latency
- **Buffer Underrun Prevention**: The audio callback must execute within its strict time budget (under 3ms per 128-sample block) to prevent audio artifacts.
- **Memory Management**: Zero dynamic memory allocations (`malloc`/`new`) inside the real-time audio loop to avoid garbage collection spikes.

## 10. Main Entities / Data Model

### SynthPatch (Configuration)
- **OscillatorType**: Enum (Sine, Square, Sawtooth, Triangle).
- **Attack, Decay, Sustain, Release**: Floats (seconds / gain).
- **CutoffFrequency, Resonance**: Floats.

## 11. System Components

- **WASM DSP Module**: Core math and audio generation compiled from C/C++/Rust.
- **AudioWorklet Processor**: Browser audio bridge running the audio rendering loop.
- **UI Frontend**: React/Vue/Vanilla JS interface with interactive controls.

## 12. Important Technical Challenges

### Real-Time Audio Constraints
- **Challenge**: The audio thread cannot wait for locks, file I/O, or memory allocations. Any delay results in audible pops or clicks.
- **Concepts**: Lock-free programming, fixed-size ring buffers, stack allocation.

### Inter-Thread Communication
- **Challenge**: Sending parameter changes from the main UI thread to the audio worklet thread must be thread-safe and non-blocking.
- **Concepts**: `Atomics`, `SharedArrayBuffer`, MessagePort messaging.

## 13. Suggested Technology Areas

- **DSP Core**: Rust (`wasm-pack`) or C++ (`emscripten`).
- **Frontend**: HTML5 Canvas, Web Audio API, TypeScript/JavaScript.

## 14. Skills and Knowledge Gained

### Digital Signal Processing (DSP)
- Waveform generation and oscillator math.
- Filter design and envelope shaping.

### WebAssembly & Concurrency
- Compiling low-level code to WebAssembly.
- Writing real-time audio worklets with strict performance budgets.

## 15. Recommended Development Phases

1. **Phase 1 - WASM Oscillator**: Write a simple C++/Rust function that generates a sine wave buffer and compile it to WASM.
2. **Phase 2 - AudioWorklet Hookup**: Load the WASM module inside an `AudioWorkletProcessor` and play a continuous tone through speakers.
3. **Phase 3 - ADSR & Filters**: Add envelope shaping and filter algorithms to the DSP core.
4. **Phase 4 - Interactive UI**: Build the keyboard and parameter knobs on the frontend to control sound properties live.

## 16. Testing Requirements

- **Unit Tests**: Test DSP math functions (e.g., oscillator frequency calculation, envelope curves) in isolation before WASM compilation.
- **Performance Profiling**: Use browser developer tools to verify that the audio worklet callback executes comfortably within the frame budget.

## 17. Security Considerations

- **SharedArrayBuffer Security**: Browsers require specific HTTP headers (`Cross-Origin-Opener-Policy` and `Cross-Origin-Embedder-Policy`) to enable high-resolution timers and `SharedArrayBuffer`. Configure local dev servers correctly.

## 18. Possible Extensions

- **Polyphony**: Extend the synthesizer engine to support multiple simultaneous notes (polyphonic chords) with voice stealing.
- **Delay / Reverb Effects**: Implement custom delay line and feedback loop effects in WASM.

## 19. Learning Questions

- Why are dynamic memory allocations forbidden inside a real-time audio render loop?
- How does an AudioWorklet differ from a standard Web Worker?
- What is the mathematical difference between a Sawtooth wave and a Square wave in terms of harmonic overtones?

## 20. Completion Criteria

- [ ] WASM module successfully compiles and loads in the browser.
- [ ] AudioWorklet streams real-time audio generated by the WASM module.
- [ ] User can play notes using the virtual keyboard without audio stuttering or clicking.
- [ ] ADSR envelope and filter controls alter the sound characteristics in real time.
- [ ] Zero memory allocations occur inside the audio processing loop.