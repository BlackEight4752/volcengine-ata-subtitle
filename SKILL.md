---
name: volcengine-ata-subtitle
description: "Generate subtitles with automatic time alignment using Volcengine ATA API. Use when the user wants to: (1) add time-aligned subtitles to videos, (2) convert text + audio to SRT/ASS format, or (3) automate subtitle creation workflow."
version: 1.0.0
category: media-processing
argument-hint: "[audio file] [text file] [output file]"
---

# Volcengine ATA Subtitle (自动打轴)

Generate subtitles with automatic time alignment using Volcengine's ATA (Automatic Time Alignment) API.

## Prerequisites

Set the following environment variables or create a config file:

### Option A: Environment Variables

```bash
export VOLC_ATA_APP_ID="your-app-id"
export VOLC_ATA_TOKEN="your-access-token"
export VOLC_ATA_API_BASE="https://openspeech.bytedance.com"
```

### Option B: Config File

Create `~/.volcengine_ata.conf`:

```ini
[credentials]
appid = your-app-id
access_token = your-access-token
secret_key = your-secret-key

[api]
base_url = https://openspeech.bytedance.com
submit_path = /api/v1/vc/ata/submit
query_path = /api/v1/vc/ata/query
```

## Execution (Python CLI Tool)

A Python CLI tool is provided at `~/.openclaw/workspace/skills/volcengine-ata-subtitle/volc_ata.py`.

### Quick Examples

```bash
# Basic usage: audio + text → SRT subtitle
python3 ~/.openclaw/workspace/skills/volcengine-ata-subtitle/volc_ata.py \
  --audio storage/audio.wav \
  --text storage/subtitle.txt \
  --output storage/subtitles/final.srt

# Specify output format (srt or ass)
python3 ~/.openclaw/workspace/skills/volcengine-ata-subtitle/volc_ata.py \
  --audio storage/audio.wav \
  --text storage/subtitle.txt \
  --output storage/subtitles/final.ass \
  --format ass

# With custom API configuration
python3 ~/.openclaw/workspace/skills/volcengine-ata-subtitle/volc_ata.py \
  --audio storage/audio.wav \
  --text storage/subtitle.txt \
  --output storage/subtitles/final.srt \
  --app-id 2942487587 \
  --token your-token
```

## Workflow Integration

### Full Subtitle Workflow

```bash
cd /vol2/1000/nvme/OpenClaw/double-dog-radio

# 1. Extract audio from video (16kHz mono PCM)
ffmpeg -i videos/batch-001/video-raw.mp4 \
  -vn -acodec pcm_s16le -ar 16000 -ac 1 \
  storage/audio.wav -y

# 2. Prepare subtitle text (plain text, one sentence per line)
cat > storage/subtitle.txt << 'TXT'
第一句字幕
第二句字幕
第三句字幕
TXT

# 3. Generate time-aligned subtitles
python3 ~/.openclaw/workspace/skills/volcengine-ata-subtitle/volc_ata.py \
  --audio storage/audio.wav \
  --text storage/subtitle.txt \
  --output storage/subtitles/final.srt

# 4. Add subtitles to video
ffmpeg -i videos/batch-001/video-raw.mp4 \
  -vf "subtitles=storage/subtitles/final.srt:force_style='FontName=Arial,FontSize=16,PrimaryColour=&HFFFFFF,OutlineColour=&H000000,Outline=2,Shadow=0,MarginV=20'" \
  -c:a copy videos/batch-001/video-final.mp4
```

### One-Click Script

Create `scripts/full-subtitle.sh`:

```bash
#!/bin/bash
set -e

VIDEO=$1
TEXT_FILE=$2
OUTPUT=$3

echo "🎬 Starting subtitle workflow..."

# 1. Extract audio
echo "📹 Extracting audio..."
ffmpeg -i "$VIDEO" -vn -acodec pcm_s16le -ar 16000 -ac 1 \
  storage/temp-audio.wav -y

# 2. ATA time alignment
echo "🔥 Running ATA..."
python3 ~/.openclaw/workspace/skills/volcengine-ata-subtitle/volc_ata.py \
  --audio storage/temp-audio.wav \
  --text "$TEXT_FILE" \
  --output storage/temp-subtitle.srt

# 3. Add subtitles
echo "🎨 Adding subtitles..."
ffmpeg -i "$VIDEO" \
  -vf "subtitles=storage/temp-subtitle.srt:force_style='FontName=Arial,FontSize=16,PrimaryColour=&HFFFFFF,OutlineColour=&H000000,Outline=2,Shadow=0,MarginV=20'" \
  -c:a copy "$OUTPUT"

# 4. Cleanup
rm -f storage/temp-audio.wav storage/temp-subtitle.srt

echo "✅ Done! Output: $OUTPUT"
```

## Input Requirements

### Audio File

- **Format**: WAV (PCM)
- **Sample Rate**: 16000 Hz (16kHz)
- **Channels**: 1 (mono)
- **Encoding**: 16-bit PCM (`pcm_s16le`)

**Extract from video**:
```bash
ffmpeg -i input.mp4 -vn -acodec pcm_s16le -ar 16000 -ac 1 audio.wav
```

### Text File

- **Format**: Plain text (UTF-8)
- **Structure**: One sentence per line
- **No punctuation**: ATA will handle automatically
- **No timestamps**: Pure text only

**Example**:
```
主人闹钟没响睡过头了
我们俩轮流用鼻子拱他脸
他以为地震了抱着枕头就跑
```

## Output Formats

### SRT (SubRip)

```srt
1
00:00:00,000 --> 00:00:02,500
第一句字幕

2
00:00:02,500 --> 00:00:05,000
第二句字幕
```

### ASS (Advanced Substation Alpha)

```ass
[Script Info]
Title: ATA Subtitles
ScriptType: v4.00+

[Events]
Format: Layer, Start, End, Style, Name, MarginL, MarginR, MarginV, Effect, Text
Dialogue: 0,0:00:00.00,0:00:02.50,Default,,0,0,0,,第一句字幕
```

## API Reference

### Submit Task

```bash
curl -X POST "https://openspeech.bytedance.com/api/v1/vc/ata/submit" \
  -H "Authorization: Bearer; YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "app": {
      "appid": "YOUR_APP_ID"
    },
    "audio": "BASE64_ENCODED_AUDIO",
    "text": "SUBTITLE_TEXT",
    "format": "srt"
  }'
```

### Query Result

```bash
curl -X POST "https://openspeech.bytedance.com/api/v1/vc/ata/query" \
  -H "Authorization: Bearer; YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "TASK_ID"
  }'
```

## Rules

1. **Always check** that credentials are configured before making API calls.
2. **Audio must be 16kHz mono PCM** - convert if necessary with ffmpeg.
3. **Text should be plain** - no timestamps, no punctuation.
4. **Poll interval**: 5 seconds between status checks.
5. **Default format**: SRT (most compatible).
6. **Handle errors gracefully** - display clear error messages.
7. **Clean up temp files** after workflow completion.

## Troubleshooting

### Invalid Sample Rate

**Error**: `Invalid sample rate, expected 16000Hz`

**Fix**:
```bash
ffmpeg -i input.mp4 -ar 16000 -ac 1 audio.wav
```

### Authorization Failed

**Error**: `Authorization failed`

**Fix**: Check token format. Should be `Bearer; {token}` (with semicolon).

### Subtitle Timing Off

**Fix**: Adjust all timestamps in SRT file using subtitle editor.

## Related Documents

- [SUBTITLE-WORKFLOW.md](../../double-dog-radio/SUBTITLE-WORKFLOW.md) - Full workflow guide
- [Volcengine ATA Docs](https://www.volcengine.com/docs/4761/102177) - Official documentation
