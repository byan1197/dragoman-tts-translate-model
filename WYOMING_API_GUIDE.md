# Wyoming Faster-Whisper API Guide

## Overview

This document describes how to use the faster-whisper Docker container for speech-to-text transcription using the Wyoming protocol.

**Server Details:**
- Host: `localhost`
- Port: `10300`
- Protocol: TCP socket with newline-delimited JSON + binary payloads
- Model: `tiny.en` (English-only, fastest)

## Wyoming Protocol Specification

### Message Format

Messages are newline-delimited JSON (`\n` terminated). Some messages include binary payloads that follow immediately after the newline.

**Message Structure:**
```json
{
  "type": "message_type",
  "data": { /* message-specific data */ },
  "data_length": 1234  // Optional: indicates binary payload follows (in bytes)
}
```

When `data_length` is present, exactly that many bytes of binary data follow the JSON line.

### Message Flow for Transcription

```
Client                          Server
  |                               |
  |--- describe ----------------→ |  (optional, to get server info)
  | ←------ info + binary --------|
  |                               |
  |--- transcribe --------------→ |  (start transcription request)
  |--- audio-start -------------→ |  (audio format specification)
  |--- audio-chunk + audio -----→ |  (can send multiple chunks)
  |--- audio-chunk + audio -----→ |
  |--- audio-stop --------------→ |  (end of audio)
  |                               |
  | ←------ transcript + JSON ----|  (result in binary payload)
```

## Audio Requirements

### Supported Format
- **Sample Rate:** 16000 Hz (16 kHz)
- **Bit Depth:** 16-bit (width: 2 bytes)
- **Channels:** 1 (mono)
- **Encoding:** PCM signed 16-bit little-endian (pcm_s16le)

### Converting Audio Files

If you have a WAV file, remove the 44-byte header before sending:
```javascript
const audioData = fs.readFileSync('audio.wav').slice(44);
```

Or use FFmpeg to convert any audio to the correct format:
```bash
ffmpeg -i input.mp3 -ar 16000 -ac 1 -f s16le -acodec pcm_s16le output.pcm
```

## API Messages

### 1. Describe (Optional)
Get server capabilities and info.

**Send:**
```json
{"type": "describe"}
```

**Receive:**
```json
{"type": "info", "version": "1.8.0", "data_length": 1147}
[followed by JSON payload with server info]
```

### 2. Transcribe
Start a transcription request.

**Send:**
```json
{"type": "transcribe", "data": {}}
```

### 3. Audio Start
Specify audio format parameters.

**Send:**
```json
{
  "type": "audio-start",
  "data": {
    "rate": 16000,
    "width": 2,
    "channels": 1
  }
}
```

### 4. Audio Chunk
Send audio data. Can be sent multiple times for streaming.

**Send:**
```json
{
  "type": "audio-chunk",
  "data": {
    "rate": 16000,
    "width": 2,
    "channels": 1
  },
  "payload_length": 123456
}
[followed by raw PCM audio bytes]
```

**Note:** The `payload_length` field indicates how many bytes of audio data follow.

### 5. Audio Stop
Signal end of audio stream.

**Send:**
```json
{"type": "audio-stop", "data": {}}
```

### 6. Transcript (Received)
The transcription result.

**Receive:**
```json
{"type": "transcript", "data_length": 404}
[followed by JSON payload]
```

**Binary Payload Structure:**
```json
{
  "text": "The transcribed text appears here..."
}
```

## Complete Working Example (Node.js)

```javascript
const net = require("net");
const fs = require("fs");

const client = new net.Socket();
const FILE_PATH = "./audio.wav"; // 16kHz mono WAV file

// Protocol state
let buffer = Buffer.alloc(0);
let waitingForBinary = false;
let expectedLength = 0;
let currentMessageType = null;

client.connect(10300, "localhost", () => {
  console.log("Connected to Wyoming server");
  startTranscription();
});

client.on("data", (data) => {
  buffer = Buffer.concat([buffer, data]);

  while (buffer.length > 0) {
    if (!waitingForBinary) {
      // Parse JSON line
      const newlineIndex = buffer.indexOf('\n');
      if (newlineIndex === -1) break;

      const line = buffer.slice(0, newlineIndex).toString();
      buffer = buffer.slice(newlineIndex + 1);

      const msg = JSON.parse(line);
      console.log("Received:", msg.type);

      if (msg.data_length) {
        waitingForBinary = true;
        expectedLength = msg.data_length;
        currentMessageType = msg.type;
      }

      if (msg.type === "transcript" && !msg.data_length) {
        console.log("Transcript:", msg.data.text);
        client.destroy();
      }
    } else {
      // Read binary payload
      if (buffer.length >= expectedLength) {
        const binaryData = buffer.slice(0, expectedLength);
        buffer = buffer.slice(expectedLength);

        if (currentMessageType === "transcript") {
          const result = JSON.parse(binaryData.toString());
          console.log("Transcript:", result.text);
          client.destroy();
          process.exit(0);
        }

        waitingForBinary = false;
        expectedLength = 0;
        currentMessageType = null;
      } else {
        break;
      }
    }
  }
});

function startTranscription() {
  // 1. Start transcription
  client.write(JSON.stringify({ type: "transcribe", data: {} }) + "\n");

  // 2. Specify audio format
  client.write(
    JSON.stringify({
      type: "audio-start",
      data: { rate: 16000, width: 2, channels: 1 }
    }) + "\n"
  );

  // 3. Read and send audio (skip 44-byte WAV header)
  const audioData = fs.readFileSync(FILE_PATH).slice(44);

  client.write(
    JSON.stringify({
      type: "audio-chunk",
      data: { rate: 16000, width: 2, channels: 1 },
      payload_length: audioData.length
    }) + "\n"
  );
  client.write(audioData);

  // 4. End audio stream
  client.write(JSON.stringify({ type: "audio-stop", data: {} }) + "\n");
}

client.on("error", (err) => console.error("Error:", err));
```

## Python Example

```python
import socket
import json
import struct

HOST = 'localhost'
PORT = 10300

def send_message(sock, msg_type, data=None, payload=None):
    """Send a Wyoming protocol message"""
    message = {"type": msg_type}
    if data:
        message["data"] = data
    if payload:
        message["payload_length"] = len(payload)

    # Send JSON line
    sock.sendall((json.dumps(message) + "\n").encode())

    # Send binary payload if present
    if payload:
        sock.sendall(payload)

def receive_message(sock):
    """Receive a Wyoming protocol message"""
    # Read until newline
    line = b""
    while True:
        char = sock.recv(1)
        if char == b"\n":
            break
        line += char

    msg = json.loads(line.decode())

    # Read binary payload if present
    if "data_length" in msg:
        payload = sock.recv(msg["data_length"])
        return msg, payload

    return msg, None

# Example usage
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
    sock.connect((HOST, PORT))

    # Start transcription
    send_message(sock, "transcribe", data={})
    send_message(sock, "audio-start", data={
        "rate": 16000,
        "width": 2,
        "channels": 1
    })

    # Read audio file (skip 44-byte WAV header)
    with open("audio.wav", "rb") as f:
        f.seek(44)
        audio_data = f.read()

    # Send audio
    send_message(sock, "audio-chunk",
                data={"rate": 16000, "width": 2, "channels": 1},
                payload=audio_data)

    send_message(sock, "audio-stop", data={})

    # Receive transcript
    while True:
        msg, payload = receive_message(sock)
        if msg["type"] == "transcript":
            result = json.loads(payload.decode())
            print("Transcript:", result["text"])
            break
```

## Streaming Audio

For real-time transcription, send multiple `audio-chunk` messages:

```javascript
const CHUNK_SIZE = 4096; // 4KB chunks

function streamAudio(audioBuffer) {
  for (let i = 0; i < audioBuffer.length; i += CHUNK_SIZE) {
    const chunk = audioBuffer.slice(i, i + CHUNK_SIZE);

    client.write(
      JSON.stringify({
        type: "audio-chunk",
        data: { rate: 16000, width: 2, channels: 1 },
        payload_length: chunk.length
      }) + "\n"
    );
    client.write(chunk);
  }
}
```

## Common Issues

### 1. No Response from Server
- Ensure audio format matches: 16kHz, 16-bit, mono
- Check that you're sending `payload_length` field (not `data.payload_length`)
- Verify you're skipping the WAV header (44 bytes)
- Make sure to send `audio-stop` to signal completion

### 2. Connection Refused
- Check Docker container is running: `docker-compose ps`
- Verify port 10300 is exposed and not blocked
- Check logs: `docker-compose logs faster-whisper`

### 3. Empty or Garbled Transcripts
- Audio quality may be poor
- Audio format may be incorrect (use ffprobe to verify)
- Try a different Whisper model (edit docker-compose.yaml)

## Docker Container Configuration

Your current setup in `docker-compose.yaml`:
```yaml
services:
  faster-whisper:
    image: lscr.io/linuxserver/faster-whisper:latest
    platform: linux/arm64  # For Apple Silicon
    environment:
      - WHISPER_MODEL=tiny.en  # Fast but less accurate
      - WHISPER_COMPUTE_TYPE=float32
    ports:
      - 10300:10300
```

### Available Models (in order of speed/accuracy tradeoff)

- `tiny.en` - Fastest, least accurate (current)
- `base.en` - Fast, better accuracy
- `small.en` - Slower, good accuracy
- `medium.en` - Slow, very good accuracy
- `large-v3` - Slowest, best accuracy (multilingual)

## Testing

Test the server with netcat:
```bash
echo '{"type":"describe"}' | nc localhost 10300
```

You should see JSON output with server info.

## Performance Notes

- First transcription may be slower (model loading)
- The `tiny.en` model is optimized for speed over accuracy
- Transcription is NOT real-time streaming (processes complete audio)
- Expected latency: 1-3 seconds for 10-30 second audio clips

## Backend Integration Checklist

- [ ] Ensure audio is converted to 16kHz, 16-bit, mono PCM
- [ ] Implement proper Wyoming protocol message parsing
- [ ] Handle binary payloads correctly (using `data_length` field)
- [ ] Add timeout handling (30-60 seconds recommended)
- [ ] Add error handling for connection failures
- [ ] Consider connection pooling for high-volume use
- [ ] Log transcription times for monitoring

---

**Working example:** See `test-working.js` for a complete, tested implementation.
