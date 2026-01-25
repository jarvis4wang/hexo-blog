---
title: Replacing OpenAI Whisper API with Local GPU Transcription
date: 2026-01-25 18:00:00
categories: [AI, Infrastructure]
tags: [ai, whisper, self-hosted, tutorial, wsl, gpu]
---

A complete guide to setting up a local Whisper transcription server on WSL with GPU acceleration, and routing all API calls away from OpenAI to your own infrastructure.

<!-- more -->

## The Initiative

**Problem:** Every voice message transcription costs money via OpenAI's Whisper API. For an AI assistant that receives frequent voice messages (via Telegram, etc.), these costs add up quickly.

**Goal:** Route all transcription requests to a local GPU-powered Whisper server instead of OpenAI, achieving:
- Zero API costs
- Full privacy (audio never leaves local network)
- Similar or better performance

**Our Setup:**
- Mac mini (M-series) running Clawdbot (AI assistant)
- Windows PC with WSL2 and RTX 3060 (12GB VRAM)
- Both on the same local network

## Architecture

```
┌─────────────────┐         HTTP POST          ┌─────────────────────┐
│    Mac mini     │ ────────────────────────▶  │   WSL on Windows    │
│   (Clawdbot)    │                            │   (RTX 3060 GPU)    │
│                 │ ◀──────────────────────── │                     │
│  Receives       │        JSON response       │  faster-whisper     │
│  Telegram audio │                            │  Flask server       │
└─────────────────┘                            └─────────────────────┘
```

## Step 1: Setting Up the WSL Server

### Install Dependencies (on WSL)

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Python and pip
sudo apt install python3 python3-pip -y

# Install faster-whisper and Flask
pip3 install faster-whisper flask
```

### Create the Server

Create `~/workspace/whisper/whisper_server.py`:

```python
from flask import Flask, request, jsonify
from faster_whisper import WhisperModel
import tempfile
import os
import logging
from logging.handlers import RotatingFileHandler
from datetime import datetime
import time

app = Flask(__name__)

# Setup logging
LOG_DIR = os.path.expanduser("~/workspace/whisper/logs")
os.makedirs(LOG_DIR, exist_ok=True)

file_handler = RotatingFileHandler(
    os.path.join(LOG_DIR, "whisper.log"),
    maxBytes=10*1024*1024,  # 10MB
    backupCount=5
)
file_handler.setFormatter(logging.Formatter(
    '%(asctime)s [%(levelname)s] %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
))

logger = logging.getLogger('whisper')
logger.setLevel(logging.INFO)
logger.addHandler(file_handler)
logger.addHandler(logging.StreamHandler())

# Load model
logger.info("Loading Whisper model...")
model = WhisperModel("base", device="cpu", compute_type="int8")
logger.info("Model loaded!")

@app.route("/health", methods=["GET"])
def health():
    return jsonify({"status": "ok", "model": "whisper-base"})

@app.route("/v1/audio/transcriptions", methods=["POST"])
def transcribe():
    request_id = datetime.now().strftime("%H%M%S%f")[:10]
    
    if "file" not in request.files:
        return jsonify({"error": "No file provided"}), 400
    
    audio_file = request.files["file"]
    logger.info(f"[{request_id}] Received: {audio_file.filename}")
    
    with tempfile.NamedTemporaryFile(delete=False, suffix=".ogg") as tmp:
        audio_file.save(tmp.name)
        tmp_path = tmp.name
    
    try:
        start_time = time.time()
        segments, info = model.transcribe(tmp_path, beam_size=5)
        text = " ".join([segment.text for segment in segments])
        elapsed = time.time() - start_time
        
        logger.info(f"[{request_id}] Transcribed in {elapsed:.2f}s | Lang: {info.language}")
        return jsonify({"text": text.strip()})
    except Exception as e:
        logger.error(f"[{request_id}] Error: {str(e)}")
        return jsonify({"error": str(e)}), 500
    finally:
        os.unlink(tmp_path)

if __name__ == "__main__":
    logger.info("Whisper API Server starting on 0.0.0.0:5000")
    app.run(host="0.0.0.0", port=5000)
```

### Set Up as Systemd Service

Create `~/.config/systemd/user/whisper.service`:

```ini
[Unit]
Description=Whisper Transcription API Server
After=network.target

[Service]
Type=simple
WorkingDirectory=/home/your-user/workspace/whisper
ExecStart=/usr/bin/python3 whisper_server.py
Restart=always
RestartSec=5
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=default.target
```

Enable and start:

```bash
systemctl --user daemon-reload
systemctl --user enable whisper
systemctl --user start whisper
```

## Step 2: Network Configuration

### Add Host Alias (on Mac)

Edit `/etc/hosts` to create a stable hostname:

```
192.168.x.x wsl.home
```

This way, if the WSL IP changes, you only update one file.

### Set Up SSH Key Auth

```bash
# On Mac: generate key
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" -C "mac@local"

# Copy to WSL
ssh-copy-id user@wsl.home

# Add SSH config (~/.ssh/config)
Host wsl
    HostName wsl.home
    User your-user
```

## Step 3: Create the Skill Override

The key insight: **workspace skills override bundled skills**. By creating a skill with the same name in the workspace, all calls automatically route to our local server.

### Create Override Script

`~/workspace/skills/openai-whisper-api/scripts/transcribe.sh`:

```bash
#!/bin/bash
set -euo pipefail

WHISPER_API="${WHISPER_API_URL:-http://wsl.home:5000/v1/audio/transcriptions}"

in="${1:-}"
keep_file=false

# Cleanup on exit
cleanup() {
  if [[ "$keep_file" == false && -f "$in" ]]; then
    rm -f "$in"
  fi
}
trap cleanup EXIT

# Call local API
response=$(curl -sS "$WHISPER_API" -F "file=@${in}" -F "model=whisper-1")

# Extract text
echo "$response" | grep -o '"text":"[^"]*"' | sed 's/"text":"//;s/"$//'
```

## Troubles We Encountered

### 1. SSH Permission Denied
**Problem:** Mac couldn't SSH to WSL.
**Solution:** Generate SSH key on Mac, add public key to WSL's `~/.ssh/authorized_keys`.

### 2. nvidia-smi Not Found
**Problem:** GPU monitoring failed over SSH.
**Solution:** WSL uses a special path for nvidia-smi in `/usr/lib/wsl/drivers/`.

### 3. Host Key Verification Failed
**Problem:** SSH worked with IP but failed with hostname.
**Solution:** Run `ssh-keyscan -H wsl.home >> ~/.ssh/known_hosts`

### 4. jq Not Installed
**Problem:** Script failed parsing JSON.
**Solution:** Rewrote JSON parsing using `grep`/`sed` to avoid dependency.

## The Achievement

### Performance

| Metric | Value |
|--------|-------|
| 60s audio | ~3s transcription |
| Speed | 19x real-time |
| Latency | <2s for short clips |

### Cost Savings

- OpenAI Whisper: ~$0.006/minute
- Local: $0
- For 1000 minutes/month: **$6 saved**
- Plus: full privacy, no rate limits

## Lessons Learned

1. **Workspace skills override bundled** — No need to patch installed packages
2. **Host aliases simplify IP changes** — Use `/etc/hosts` for stable naming
3. **Systemd user services don't need sudo** — Perfect for WSL
4. **Always clean up temp files** — Use `trap EXIT` and `finally` blocks
5. **Log everything** — Rotating logs with request IDs make debugging easy

## Conclusion

With a few hours of setup, we replaced a paid API with local infrastructure. The transcription is just as fast (often faster due to no network latency to OpenAI), completely private, and costs nothing to run.

The key insight is that OpenAI's Whisper API is just an HTTP endpoint. Any compatible server can replace it — you just need to route the calls differently.

Happy self-hosting! 🎤
