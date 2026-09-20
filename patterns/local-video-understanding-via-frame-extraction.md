---
type: pattern
date: "2026-08-08"
source: The personal agent local video understanding smoke test, Qwen3-VL-4B on a 16GB MacBook Pro, frame extraction via ffmpeg.
status: validated
tags:
  - local-models
  - video-understanding
  - performance
  - memory
---

# Local Video Understanding Via Frame Extraction

Local Vision-Language Models (VLMs) like Qwen2-VL or Qwen2.5-VL running under local runtimes (LM Studio, Ollama) do not ingest raw video files natively through standard OpenAI-compatible `/v1/chat/completions` API endpoints. Instead, video understanding must be achieved by extracting, downscaling, and transmitting a sequence of image frames.

Without careful framing, this process will easily thrash model memory, overflow the context window, and cause silent failure.

## The Pattern

To perform local video understanding reliably on resource-constrained hardware:

1. **Extract a Constant Number of Frames (Bounded Size):**
   Do not extract frames at a constant frame rate (e.g., 1 fps) for arbitrary videos. A 10-minute video at 1 fps produces 600 frames, which will blow the context window and crash the runtime. Instead, compute the video duration first using `ffprobe` and extract a **constant maximum number of frames** (e.g., 10 to 15 frames) evenly distributed across the duration.
   
2. **Scale Down Prior to Base64 Encoding:**
   Vision models tile images into visual tokens. Large images (like 4K screenshots or phone photos) translate into thousands of tokens per frame. Scale down each frame during extraction so that its **longest edge is exactly 768px**.
   
3. **Execute Extraction in a Single ffmpeg Seek Pass:**
   Rather than piping or running complex filter graphs, run a fast seek command per frame to write a temporary JPG:
   ```bash
   ffmpeg -ss <timestamp> -i <video_path> -vframes 1 -vf "scale='if(gt(iw,ih),768,-1)':'if(gt(iw,ih),-1,768)'" -q:v 2 <output_path>
   ```

4. **Multi-Image Payload Delivery:**
   Encode the resulting frames as Base64 JPEG strings and deliver them in a single `user` message with multiple `image_url` elements, alongside a chronological instructions prompt:
   ```json
   {
     "role": "user",
     "content": [
       {"type": "text", "text": "Describe what is happening in this video sequence chronologically."},
       {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,..."}},
       {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,..."}}
     ]
   }
   ```

5. **Transcribe Audio Track (Multimodal Integration):**
   If the video contains an audio stream, extract it to a mono WAV file (`ffmpeg -i video.mp4 -vn -acodec pcm_s16le -ar 16000 -ac 1 audio.wav`) and transcribe it locally using a speech-to-text model (such as the Whisper CLI: `whisper audio.wav --model base`). Append the resulting transcript text to the VLM prompt context:
   ```text
   Here is the transcript of the audio track of the video:
   "[transcript text]"
   ```

## Why this is critical for local runtimes

### The Memory Ceiling
Each frame consumes a significant slice of the context window. With a 4B VLM (such as Qwen3-VL-4B / Qwen2-VL-7B) loaded, the model's KV Cache scales linearly with the context length. 

- **Optimizing Parallelism:** Under LM Studio, load the model with `--parallel 1` instead of the default `4` to prevent multiplying the KV cache memory footprint. This keeps the VLM resident in Unified Memory and avoids disk swapping.
- **Context Ceiling:** Limit the context to `8192` or `16384` depending on available RAM. Ten 768px frames plus a prompt will easily fit inside a standard 22,784 context limit if parallelism is constrained.

### The Downscaling Math
A 5.1 MB phone photo at 3600px can take 56–132s to process or silently truncate to garbage (c.f. [A Fast Answer Is A Suspect Answer](a-fast-answer-is-a-suspect-answer.md)). Downscaling to 768px reduces the disk footprint to ~120KB and processes in less than 2s per frame, preserving all visual fidelity required for chronological activity recognition.
