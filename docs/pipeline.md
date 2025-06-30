# Audio Processing Pipeline

This document describes the internal audio pipeline used by `RealtimeSTT`. It focuses on the local recording flow implemented in `RealtimeSTT/audio_recorder.py` and related modules.

## Overview

The pipeline captures audio from a microphone, performs voice activity and wake word detection, optionally streams partial results in real time, and produces a final transcription. Processing is distributed across multiple processes and threads to minimize latency.

The main stages are:

1. **Audio Input** – Capture raw PCM samples with PyAudio.
2. **Pre–processing** – Resample and buffer audio chunks.
3. **Voice/Wake Word Detection** – Decide when to start and stop recording.
4. **Real‑Time Transcription** – Optional short‑pause updates during recording.
5. **Final Transcription** – Transcribe the recorded utterance with the main Whisper model.

A simplified diagram is shown below:

```text
Microphone -> AudioInput (PyAudio) -> Queue -> Recording Worker
    |                                     |-> Wake Word Detector (PV Porcupine/OpenWakeWord)
    |                                     |-> Voice Activity Detectors (WebRTC + Silero)
    |                                     |-> frames -> Realtime Worker (optional)
    |                                     +-> Final Transcription Worker
```

## AudioInput

`AudioInput` sets up a PyAudio stream using the best available sample rate for the selected device. It can list all input devices and choose the highest supported rate. Incoming frames are optionally low‑pass filtered and resampled to the target rate (16 kHz by default) using `scipy.signal.resample_poly` if the device operates at a different rate. Each call to `read_chunk()` returns a block of samples which are pushed into a multiprocessing queue. The default `CHUNK_SIZE` used by `AudioInput` is 1024 frames (about 64 ms at 16 kHz).

## Chunking and Buffering

Audio is handled in discrete chunks. Each chunk produced by `AudioInput` is
exactly `CHUNK_SIZE` samples long. The recorder maintains a `buffer_size`
parameter (512 by default) which controls how many samples are considered when
running voice activity detection. Chunks read from the device are appended to an
internal buffer until at least `2 * buffer_size` bytes are available. This
coalesced block is then pushed into the processing queue so that both WebRTC and
Silero VAD operate on stable, fixed-size windows. The buffer also retains a
rolling pre‑recording history so that a short span of audio prior to activation
is included in the final recording.

## Recording Worker

The recording worker thread consumes chunks from the queue. It keeps a short **pre‑recording buffer** so that speech leading up to activation is not lost. Two VAD engines work together:

- **WebRTC VAD** quickly detects speech onset and offset.
- **Silero VAD** provides more robust detection and can terminate recording after silence.

If configured, wake word detection runs on the same audio stream using either the `pvporcupine` or `openwakeword` backend. Once a wake word or initial speech is detected the recorder enters the *recording* state. Audio frames are stored in `self.frames` until silence is detected.

When recording ends the frames are concatenated, converted to float32, and sent through a `SafePipe` to the transcription process.

## Real‑Time Worker

If real‑time transcription is enabled, another thread periodically assembles the buffered frames and sends them either to a small Whisper model or to the main model. The worker returns partial text every `realtime_processing_pause` seconds (default 0.2 s) and attempts to stabilize results by comparing consecutive outputs.

Callbacks `on_realtime_transcription_update` and `on_realtime_transcription_stabilized` are invoked with the interim text.

## Transcription Worker

The transcription worker process loads a Whisper model via `faster_whisper.WhisperModel`. Audio received over the pipe is normalized (optionally) and transcribed. Results are sent back through the pipe to the recorder thread. This process runs independently from the main Python interpreter to keep GPU operations asynchronous.

The final text is post‑processed to ensure sentences begin with an uppercase letter and end with punctuation if desired. Callbacks `on_transcription_start` and `on_recording_stop` allow applications to react when transcription begins and finishes.

## Client/Server Mode

`AudioToTextRecorderClient` wraps the same pipeline but transports the audio over WebSockets to a server implemented in `RealtimeSTT_server/stt_server.py`. The server executes the same detection and transcription steps while broadcasting transcription updates back to connected clients.

## Thread and Process Layout

- **Reader Process** – Continuously reads audio from the device and feeds the queue.
- **Recording Worker** – Runs in a thread; handles VAD, buffering, and sending audio to the transcription worker.
- **Realtime Worker** – Optional thread for on‑the‑fly transcription.
- **Transcription Worker** – Separate process hosting the Whisper model.

This multi‑process design reduces latency while keeping the main thread responsive.

## Parallelism and Scaling

The separation into processes and threads allows the CPU and GPU workloads to
run concurrently. Audio capture and VAD run in the **Reader** and **Recording**
threads while the Whisper model executes in a dedicated process. Communication
between these components happens through `SafePipe` and multiprocessing queues
which avoid the Global Interpreter Lock. On systems with multiple GPUs the
transcription worker can be launched on different devices, enabling several
recordings to be processed simultaneously from separate Python threads. The
server mode further exposes a `--batch` parameter that controls how many audio
chunks are run through the model in parallel, trading memory usage for higher
throughput when handling many clients.
